# coding: utf-8
require 'bundler/setup' unless defined?(Bundler)

require 'faraday'
require 'json'
require 'logger'
require 'mackerel/client'
require 'parallel'

class Mackerel::Client
  # defs: [ def, ... ]
  # def: { name: String, displayName: Maybe[String], unit: Maybe[String], metrics: Array[metric] }
  # metric: { name: String, displayName: Maybe[String], isStacked: Boolean }
  def define_graphs(defs)
    res = client.post("/api/v0/graph-defs/create") do |req|
      req.headers['X-Api-Key'] = @api_key
      req.headers['Content-Type'] = 'application/json'
      req.body = defs.to_json
    end

    raise "POST /api/v0/graph-defs/create failed: #{res.status} #{res.body}" unless res.success?

    JSON.parse(res.body)
  end
end

def logger
  @logger ||= Logger.new(STDERR)
end

def extract_score(res)
  return nil unless res.success?
  data = JSON.parse(res.body)
  data['lighthouseResult']['categories']['performance']['score']
end

def as_metric_name(str)
  str.gsub(/[^a-zA-Z0-9_\-]/, '-')
end

# => [url: String, score: Integer]
def fetch_psi_scores(urls, api_key)
  logger.info "Fetch scores"
  client = Faraday.new(url: 'https://www.googleapis.com')
  results = Parallel.map(urls, in_threads: 4) {|url|
    logger.info "Fetch score of #{url}"
    params = {
      url: url,
      fields: "lighthouseResult/categories/*/score",
      strategy: "desktop",
      category: "performance",
    }
    params[:key] = api_key if api_key
    [
      url,
      client.get('/pagespeedonline/v5/runPagespeed', params).tap { logger.info "Done: #{url}" }
    ]
  }
  results.map {|(url, res)| [url, extract_score(res)] }
end

def post_scores_to_mackerel(mackerel_service: , api_key: , scores: )
  logger.info "Post scores to Mackerel"
  now = Time.now
  mackerel_client = Mackerel::Client.new(mackerel_api_key: api_key)
  mackerel_client.define_graphs([
    {
      name: 'custom.pagespeed',
      unit: 'float',
      metrics: (scores.map {|(url, _)| [url, as_metric_name(url)] }.map {|(url, safe_url)|
        {
          name: "custom.pagespeed.#{safe_url}",
          displayName: url,
          isStacked: false,
        }
      }),
    },
  ])
  mackerel_client.post_service_metrics(mackerel_service, scores.map {|(url, score)|
    {
      name: "custom.pagespeed.#{as_metric_name(url)}",
      value: score * 100,
      time: now.to_i,
    }
  })
end

task :clock do
  interval_secs = Integer(ENV['INTERVAL_SECS'])
  wait_secs = 10
  logger.info "Run with interval: #{interval_secs}"
  last_run = nil
  loop do
    if last_run.nil? || (Time.now - last_run) >= interval_secs
      logger.info "run"
      last_run = Time.now
      system 'bundle', 'exec', 'rake', 'post'
    else
      logger.info "Wait #{wait_secs} ..."
      sleep wait_secs
    end
  end
end

task :kick do
  # テスト目的
  logger.info 'kicking PSI'
  google_api_key   = ENV['GOOGLE_APIKEY'] # optional
  urls             = ENV['URLS'].split(/\s/)
  scores = fetch_psi_scores(urls, google_api_key).select {|(url, score)| score}
  p scores
end

task :post do
  logger.info 'rake run'

  api_key          = ENV['MACKEREL_APIKEY'] or abort "MACKEREL_APIKEY required"
  mackerel_service = ENV['MACKEREL_SERVICE'] or abort "MACKEREL_SERVICE required"
  google_api_key   = ENV['GOOGLE_APIKEY'] # optional
  urls             = ENV['URLS'].split(/\s/)

  scores = fetch_psi_scores(urls, google_api_key).select {|(url, score)| score}
  if scores.length > 0
    post_scores_to_mackerel(mackerel_service: mackerel_service, api_key: api_key, scores: scores)
  end
end
