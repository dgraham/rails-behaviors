require 'rubygems'
require 'rake/clean'
require 'rubygems/package_task'

require 'sprockets'
require 'coffee-script'
require 'uglifier'

root = File.expand_path("..", __FILE__)
Assets = Sprockets::Environment.new(root) do |env|
  env.append_path "lib"
end

CLEAN.include "dist/*"
CLEAN.include "lib/rails/*.js"

class IgnoreDirectives < Sprockets::DirectiveProcessor
  def evaluate(context, locals, &block)
    result = super
    context.instance_variable_set(:'@directives', @directives)
    result
  end

  # Don't run any directives
  def process_directives
  end
end

class InjectDirectives < Tilt::Template
  def prepare
  end

  def evaluate(context, locals, &block)
    directives = context.instance_variable_get(:'@directives')
    directives = directives.map { |lineno, name, arg| "//= #{name} #{arg}\n" }.join
    directives + data
  end
end

task :build do
  Assets.unregister_processor('application/javascript', Sprockets::DirectiveProcessor)
  Assets.register_processor('application/javascript', IgnoreDirectives)
  Assets.register_postprocessor('application/javascript', InjectDirectives)

  Dir["#{root}/lib/rails/*.coffee"].each do |file|
    output = file.sub(/\.coffee$/, '.js')
    Assets[file].write_to(output)
  end
end

desc "Build the gem"
task :gem => :build do
  abort "Only build the gem with 1.8" unless RUBY_VERSION == '1.8.7'

  spec = Gem::Specification.load('rails-behaviors.gemspec')

  # Mark coffee-script as a development depedency
  coffee = spec.dependencies.detect { |dep| dep.name == 'coffee-script' }
  coffee.instance_variable_set :'@type', :development

  # Only include Ruby and JS files (skip Coffee)
  spec.files = Dir["README.md", "lib/**/*.{rb,js}"]

  Gem::Builder.new(spec).build
end

task :dist => :build do
  FileUtils.mkdir_p 'dist/'
  Assets['rails.js'].write_to('dist/rails.js')
  Assets.js_compressor = Uglifier.new
  Assets['rails.js'].write_to('dist/rails.min.js')
end

task :default => :dist

task :test => :build do
  Dir.chdir File.dirname(__FILE__)

  pid = fork do
    exec 'rackup', '-p', '3000', './test/config.ru'
  end

  sleep 2

  system 'open', 'http://localhost:3000/'

  sleep 3

  Process.kill 'SIGINT', pid
  Process.wait pid
end
