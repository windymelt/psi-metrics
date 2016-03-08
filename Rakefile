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
  data['ruleGroups']['SPEED']['score']
end

def as_metric_name(str)
  str.gsub(/[^a-zA-Z0-9_\-]/, '-')
end

# => [url: String, score: Integer]
def fetch_psi_scores(urls)
  logger.info "Fetch scores"
  client = Faraday.new(url: 'https://www.googleapis.com')
  results = Parallel.map(urls, in_threads: 4) {|url|
    logger.info "Fetch score of #{url}"
    [
      url,
      client.get('/pagespeedonline/v2/runPagespeed', url: url).tap { logger.info "Done: #{url}" }
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
      value: score,
      time: now.to_i,
    }
  })
end

task :post do
  logger.info 'rake run'

  api_key          = ENV['MACKEREL_APIKEY'] or abort "MACKEREL_APIKEY required"
  mackerel_service = ENV['MACKEREL_SERVICE'] or abort "MACKEREL_SERVICE required"
  urls             = ENV['URLS'].split(/\s/)

  scores = fetch_psi_scores(urls)
  post_scores_to_mackerel(mackerel_service: mackerel_service, api_key: api_key, scores: scores)
end
