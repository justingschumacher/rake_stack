#!/usr/bin/env ruby

require 'rake'
require 'aws-sdk'
require 'fileutils'
require 'logger'
require 'json'

@config_filename = 'config.yml'
@template_filename = 'template.json'
@parameters_filename = 'parameters.yml'
@mappings_filename = 'mappings.json'
@stack_filename = 'stack.yml'
@log_filename = 'aws-sdk.log'

@config = YAML.load_file(@config_filename)
AWS.config(:access_key_id     => @config[:access_key_id], 
           :secret_access_key => @config[:secret_access_key],
           :logger => Logger.new(@log_filename),
           :log_formatter => AWS::Core::LogFormatter.colored)

region = @config[:region] || 'us-east-1'
cloud_formation_endpoint = "cloudformation.#{region}.amazonaws.com"

@cfm = AWS::CloudFormation.new :max_retries => 10,
                               :cloud_formation_endpoint => cloud_formation_endpoint

def params
  credentials = AWS.config.credentials
  parameters = Hash[*YAML.load(File.open(@parameters_filename).read).map(&:to_a).flatten]
  parameters['AWSAccessKey']       = @config[:access_key_id]     if parameters.has_key? 'AWSAccessKey'
  parameters['AWSSecretAccessKey'] = @config[:secret_access_key] if parameters.has_key? 'AWSSecretAccessKey'
  return parameters
end

task :default => [:validate]

task :validate do
  desc "Validate the stack.template with CloudFormation."
  @cfm.validate_template(File.open(@template_filename).read)
  puts "CloudFormation Template Validated!"
end

task :merge do
  desc "Merge Amazon Linux AMI mappings into #{@template_filename}"
  mappings = JSON.parse(File.open(@mappings_filename).read)
  template = JSON.parse(File.open(@template_filename).read)
  template['Mappings'] = mappings
  template = JSON.pretty_generate(template)
  if @cfm.validate_template(template)
    File.open(@template_filename, 'w') { |f| f.puts template }
  end
  puts "Mappings merged into CloudFormation Template!"
end

task :parameters do
  desc "Create parameters.yml from #{@template_filename} Parameters."
  template = JSON.parse(File.open(@template_filename).read)
  parameters = template['Parameters'].collect{ |k,v| {k => v['Default']} }.to_yaml
  File.open(@parameters_filename, 'w') { |f| f.puts parameters }
  puts "CloudFormation Template Parameters Grokked!"
end

task :create  => [:merge, :validate] do 
  desc "Merge mappings #{@template_filename}."
  name = "stack-#{DateTime.now.strftime("%s")}"
  template = File.open(@template_filename).read
  begin
    stack = @cfm.stacks.create( name, template, 
      { :parameters => params,
        :capabilities => %w{CAPABILITY_IAM},
        :disable_rollback => true } )
  rescue AWS::CloudFormation::Errors::ValidationError => e
    puts e.message
  end
  File.open( @stack_filename, 'w' ){ |file|
    YAML.dump({:stack_name => stack.name}, file)
  }
  puts "Run! stack: #{stack.name}"
end

task :update => [:merge, :validate] do
  desc "Update the CloudFormation Stack"
  template = File.open(@template_filename).read
  stack_name = YAML.load_file(@stack_filename)[:stack_name]
  begin
    stack = @cfm.stacks[stack_name].update({:template => template, :parameters => params})
    puts "Update! stack #{stack_name}"
  rescue AWS::CloudFormation::Errors::ValidationError => e
    puts e.message
  end
end

task :status do
  desc "Describe the status of the CloudFormation Stack"
  template = File.open(@template_filename).read
  stack_name = YAML.load_file(@stack_filename)[:stack_name]
  begin
    if @cfm.stacks[stack_name].exists?
      status = @cfm.stacks[stack_name].status
      puts "Polled! stack is #{status}"
    else
      puts "Polled! stack does not exist anymore."
    end
  rescue AWS::CloudFormation::Errors::ValidationError => e
    puts e
  end
end

task :delete do
  desc "Delete the CloudFormation Stack"
  template = File.open(@template_filename).read
  stack_name = YAML.load_file(@stack_filename)[:stack_name]
  begin
    @cfm.stacks[stack_name].delete
    puts "Deleted! stack #{stack_name}"
  rescue AWS::CloudFormation::Errors::ValidationError => e
    puts e
  end
end