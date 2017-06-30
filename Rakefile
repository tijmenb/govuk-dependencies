require 'yaml'
require 'json'
require 'http'

Version = Struct.new(:name, :version, :name_and_version)

def counts(array)
  array.each_with_object(Hash.new(0)) { |el, counts| counts[el] += 1 }
end

class Gemfiles
  def self.all
    Dir.glob("cache/gemfiles/*").map do |filename|
      appname = filename.gsub("cache/gemfiles/", "")

      file = File.read(filename)
      lockfile = Bundler::LockfileParser.new(file)

      [appname, lockfile]
    end
  end
end

class Dependency
  attr_reader :name, :depended_on_directly

  def initialize(data)
    @name = data["id"]
    @depended_on_directly = data["depended_on_directly"]
  end

  def self.find(name)
    all.find { |gem| gem.name == name }
  end

  def self.all
    data = JSON.parse(File.read('public/matrix.json'))
    data['gems'].map { |gem_data| Dependency.new(gem_data) }
  end
end

class Application
  attr_reader :name

  def initialize(data)
    @name = data["id"]
  end

  def self.all
    data = JSON.parse(File.read('public/matrix.json'))
    data['applications'].map { |app_data| Application.new(app_data) }
  end
end

desc "Export the matrix as JSON for the network visualisation"
task :export_network do
  output = { nodes: [], links: [] }

  matrix = JSON.parse(File.read("public/matrix.json"))

  matrix["applications"].each do |application|
    output[:nodes] << {
      id: application.fetch("id"),
      group: 'applications',
      dependency_count: application["direct_dependencies"].size,
    }

    application["direct_dependencies"].each do |gem_name, attrs|
      output[:links] << { source: application["id"], target: gem_name }
    end
  end

  matrix["gems"].each do |attrs|
    next unless attrs["depended_on_directly"].size > 0
    output[:nodes] << { id: attrs["id"], group: 'gems', usage_count: attrs["depended_on_directly"].size }
  end

  File.write("public/network.json", JSON.pretty_generate(output))
end

task :export_versions do
  direct_dependencies = []

  Gemfiles.all.each do |appname, lockfile|
    lockfile.dependencies.map do |_, d|
      spec = lockfile.specs.find { |s| s.name == d.name }
      direct_dependencies << Version.new(spec.name, spec.version, spec.to_s)
    end
  end

  output = []

  direct_dependencies.uniq(&:name).sort_by(&:name).each do |app|
    versions = counts(direct_dependencies.select { |v| v.name == app.name }.map(&:version).sort.map(&:to_s))
    output << { gem_name: app.name, versions: versions}
  end

  File.write("public/versions.json", JSON.pretty_generate(output))
end

task :export_fragmentation do
  direct_dependencies = []

  Gemfiles.all.each do |appname, lockfile|
    lockfile.dependencies.map do |_, d|
      spec = lockfile.specs.find { |s| s.name == d.name }
      direct_dependencies << Version.new(spec.name, spec.version, spec.to_s)
    end
  end

  output = []

  direct_dependencies.uniq(&:name).each do |gem|
    versions_of_gem_in_apps = direct_dependencies.select { |v| v.name == gem.name }
    next unless versions_of_gem_in_apps.size > 1

    versions = counts(versions_of_gem_in_apps.map(&:version).sort.map(&:to_s))
    children = versions.map do |v, count|
      { name: "#{gem.name} #{v}", size: count }
    end

    output << { name: gem.name, children: children }
  end

  File.write("public/fragmentation.json", JSON.pretty_generate(name: "versions", children: output))
end

task :download do
  begin
    sh "mkdir cache/gemfiles"
    sh "rm cache/gemfiles/*"
  rescue
  end

  applications = YAML.load(HTTP.get('https://raw.githubusercontent.com/alphagov/govuk-developer-docs/master/data/applications.yml'))
  repos = applications.each do |application|
    next if application["retired"]

    repo_name = application.fetch('github_repo_name')
    url = "https://raw.githubusercontent.com/alphagov/#{repo_name}/master/Gemfile.lock"
    response = HTTP.get(url)

    if response.code == 200
      File.write("cache/gemfiles/#{repo_name}", response)
    else
      puts "Skipping #{repo_name}"
    end
  end
end

task :download_versions do
  begin
    sh "mkdir cache/ruby-versions"
    sh "rm cache/ruby-versions/*"
  rescue
  end

  applications = YAML.load(HTTP.get('https://raw.githubusercontent.com/alphagov/govuk-developer-docs/master/data/applications.yml'))
  repos = applications.each do |application|
    next if application["retired"]

    repo_name = application.fetch('github_repo_name')
    url = "https://raw.githubusercontent.com/alphagov/#{repo_name}/master/.ruby-version"
    response = HTTP.get(url)

    if response.code == 200
      File.write("cache/ruby-versions/#{repo_name}", response)
    else
      puts "Skipping #{repo_name}"
    end
  end
end

desc "Outputs all the applictions and gems they depend on, gems and apps they depend on"
task :generate_matrix do
  gems = {}
  applications = []

  Gemfiles.all.each do |app_name, lockfile|
    applications << {
      id: app_name,
      direct_dependencies: lockfile.dependencies.map { |_, d| d.name },
      dependencies: lockfile.specs.map(&:name),
    }

    lockfile.dependencies.map do |_, d|
      gems[d.name] ||= { depended_on: [], depended_on_directly: []}
      gems[d.name][:depended_on_directly] << app_name
    end

    lockfile.specs.map do |d|
      gems[d.name] ||= { depended_on: [], depended_on_directly: []}
      gems[d.name][:depended_on] << app_name
    end
  end

  gems_output = gems.map do |name, data|
    data[:id] = name
    data
  end

  output = {
    applications: applications,
    gems: gems_output,
  }

  File.write("public/matrix.json", JSON.pretty_generate(output))
end

task :rubygems_version_info do
  begin
    sh "mkdir cache/gem-versions"
    sh "rm cache/gem-versions/*"
  rescue
  end

  gems = JSON.parse(File.read("public/versions.json"))
  gems.each do |gem|
    gem_name = gem["gem_name"]
    url = "https://rubygems.org/api/v1/versions/#{gem_name}.json"
    response = HTTP.get(url)
    File.write("cache/gem-versions/#{gem_name}", response)
  end
end

task :output_age_of_lagging_dependencies do
  statuses = JSON.parse(File.read("public/versions.json"))

  Dir.glob('cache/gem-versions/*').each do |filename|
    app_name = filename.gsub('cache/gem-versions/', '')

    versions = JSON.parse(File.read("cache/gem-versions/#{app_name}"))
    latest_version = versions.first
    latest_release_date = Date.parse(latest_version["built_at"])

    status = statuses.find { |s| s["gem_name"] == app_name }

    cumul = status["versions"].flat_map do |version_number, version_count|
      this_version = versions.find { |v| v["number"] == version_number }
      next unless this_version
      this_release_date = Date.parse(this_version["built_at"])
      days_behind = (latest_release_date - this_release_date).to_i
      # puts "#{version_number} is #{days_behind} days behind #{latest_version["number"]}"
      # puts version_count
      [days_behind] * version_count.to_i
    end.compact

    next if cumul.size < 2

    average_age = cumul.reduce(&:+) / cumul.size.to_f

    puts "#{app_name} (#{cumul.size}) avg: #{average_age.to_i}, max: #{cumul.max}, min: #{cumul.min}"
  end
end

task :competitors do
  YAML.load_file("groups.yml").each do |group, gems|
    puts "\n\n# #{group}"
    d = []
    gems.each do |gem|
      deps = Dependency.find(gem)
      puts "#{gem}: #{deps.depended_on_directly}"
      d << deps.depended_on_directly
    end

    d.flatten!

    # puts "others: #{Application.all.map(&:name) - d}"
  end
end
