#!/usr/bin/env ruby

require 'rake'
require 'aws-sdk'
require 'fileutils'
require 'logger'
require 'json'
require 'find'
require 'pathname'
require 'csv'

@config_filename = 'config.yml'
@template_filename = 'template.json'
@parameters_filename = 'parameters.yml'
@mappings_filename = 'mappings.json'
@outputs_filename = 'outputs.yml'
@stack_filename = 'stack.yml'
@log_filename = 'aws-sdk.log'
@userdata_filename_prefix = 'userdata_'

@config = YAML.load_file(@config_filename)
AWS.config(:access_key_id     => @config[:access_key_id], 
           :secret_access_key => @config[:secret_access_key],
           :logger => Logger.new(@log_filename),
           :log_formatter => AWS::Core::LogFormatter.colored)

region = @config[:region] || 'us-east-1'

@cfm = AWS::CloudFormation.new( 
  :cloud_formation_endpoint => "cloudformation.#{region}.amazonaws.com",
  :max_retries => 10 )

region == 'us-east-1' ? s3_endpoint = "s3.amazonaws.com" : s3_endpoint = "s3-#{region}.amazonaws.com"
@s3 = AWS::S3.new( :s3_endpoint => s3_endpoint )

def params
  credentials = AWS.config.credentials
  parameters = Hash[*YAML.load(File.open(@parameters_filename).read).map(&:to_a).flatten]
  parameters['AWSAccessKey']       = @config[:access_key_id]     if parameters.has_key? 'AWSAccessKey'
  parameters['AWSSecretAccessKey'] = @config[:secret_access_key] if parameters.has_key? 'AWSSecretAccessKey'
  return parameters
end

# Convert a file to a JSON formatted CloudFormation EC2::Instance UserData property 
def jsonify_userdata(userdata_file_contents)
  contents_array = []
  userdata_file_contents.each_line do |line|
    contents_array << "\"" + line.chomp + "\\n\""
  end
  formatted_userdata_file_contents = contents_array.join(",")
  base = <<-EOS
{ "Fn::Base64" : { "Fn::Join" : ["", [
  #{formatted_userdata_file_contents}
] ] } }
EOS

  return base
end

task :default => [:validate]

desc "Validate the Template with CloudFormation."
task :validate do
  desc "Validate the stack.template with CloudFormation."
  @cfm.validate_template(File.open(@template_filename).read)
  puts "CloudFormation Template Validated!"
end

desc "Merge the Mapping and UserData sections."
task :merge => [:merge_mappings, :merge_user_data]

desc "Merge a Mappings section."
task :merge_mappings do
  mappings = JSON.parse(File.open(@mappings_filename).read)
  template = JSON.parse(File.open(@template_filename).read)
  template['Mappings'] = mappings
  template = JSON.pretty_generate(template)
  if @cfm.validate_template(template)
    File.open(@template_filename, 'w') { |f| f.puts template }
  end
  puts "Mappings merged into CloudFormation Template!"
end

desc "Merge resource specific Cloud-Init format UserData sections."
task :merge_user_data do
  template = JSON.parse(File.open(@template_filename).read)
  userdata_files = []
  Find.find('./') do |path|
    userdata_files << path if path.match(@userdata_filename_prefix)
    for userdata_file_name in userdata_files
      userdata_file_contents = File.open(userdata_file_name).read
      resource = userdata_file_name.split('_')[1].gsub(/\.sh/, '')
      userdata_json = jsonify_userdata userdata_file_contents
      template['Resources'][resource]['Properties']['UserData'] = JSON.parse(userdata_json)
    end
  end
  
  template = JSON.pretty_generate(template)
  
  if @cfm.validate_template(template)
    File.open(@template_filename, 'w') { |f| f.puts template }
  end
  puts "Cloud-Init UserData merged into CloudFormation Template!"
end

desc "Create parameters.yml from the template Parameters section."
task :parameters do
  template = JSON.parse(File.open(@template_filename).read)
  parameters = template['Parameters'].collect{ |k,v| {k => v['Default']} }.to_yaml
  File.open(@parameters_filename, 'w') { |f| f.puts parameters }
  puts "CloudFormation Template Parameters Grokked!"
end

desc "Create Ec2Instance.sh from the Ec2Instance UserData."
task   :userdata do
  data = JSON.parse(File.open(@template_filename).read)['Resources']['Ec2Instance']['Properties']['UserData']['Fn::Base64']['Fn::Join'][1]
  File.open('userdata_Ec2Instance.sh', 'w') { |f| f.puts data.join }
  puts "UserData!"  
end

desc "Create a CloudFormation Stack."
task :create  => [:merge, :validate] do 
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

desc "Update the CloudFormation Stack"
task :update => [:merge, :validate] do
  template = File.open(@template_filename).read
  stack_name = YAML.load_file(@stack_filename)[:stack_name]
  begin
    stack = @cfm.stacks[stack_name].update({:template => template, :parameters => params})
    puts "Update! stack #{stack_name}"
  rescue AWS::CloudFormation::Errors::ValidationError => e
    puts e.message
  end
end

desc "Describe the status of the CloudFormation Stack"
task :status do
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

desc "Delete the CloudFormation Stack"
task :delete do
  template = File.open(@template_filename).read
  stack_name = YAML.load_file(@stack_filename)[:stack_name]
  begin
    @cfm.stacks[stack_name].delete
    puts "Deleted! stack #{stack_name}"
  rescue AWS::CloudFormation::Errors::ValidationError => e
    puts e
  end
end

desc "Get the Outputs from a CloudFormation Stack"
task :outputs do
  stack_name = YAML.load_file(@stack_filename)[:stack_name]
  stack = @cfm.stacks[stack_name]
  begin
    if stack.status == 'CREATE_COMPLETE' and not stack.outputs.empty?
      outputs = Hash[stack.outputs.map {|o| [o.key, o.value] }]
      File.open( @outputs_filename, 'w' ){ |file|
        YAML.dump(outputs, file)
      }
      puts "Output! stack #{stack_name}"
      puts outputs
    else
      puts "No Outputs yet."
    end
  rescue AWS::CloudFormation::Errors::ValidationError => e
    puts e
  end
end

desc "Get Events from the CloudFormation Stack"
task :events do
  stack_name = YAML.load_file(@stack_filename)[:stack_name]
  stack = @cfm.stacks[stack_name]
  events = stack.events.collect {|e| "#{e.timestamp} #{e.resource_type} #{e.logical_resource_id} #{e.physical_resource_id} #{e.resource_status} #{e.resource_status_reason}"}
  File.open( @events_filename, 'w' ){ |file| events }
  puts events
end

desc "Stage the CloudFormation Template to the S3 Bucket"
task :stage do
  bucket_name = YAML.load_file(@config_filename)[:staging_bucket]
  prefix = YAML.load_file(@config_filename)[:staging_prefix]
  begin
    bucket = @s3.buckets[bucket_name]
    file_names = FileList.new("template.json")
    file_names.each do |file_name|
      path = Pathname.new(file_name)
      unless path.directory?
        key = "#{prefix}/#{path}"
        uri = "https://#{bucket_name}.s3.amazonaws.com/#{key}"
        puts uri
        bucket.objects[key].write(path)
      end
    end
  end
end

desc "Create the Amazon S3 Buckets"
task :buckets => [:staging_bucket, :billing_bucket]

desc "Create the Amazon S3 staging Bucket"
task :staging_bucket do
  bucket_name = YAML.load_file(@config_filename)[:staging_bucket]
  begin
    bucket = @s3.buckets.create(bucket_name)
    puts "Staging Bucket! #{bucket_name}"
  end
  begin
    policy = AWS::S3::Policy.new
    policy.allow(
      :actions => ['s3:GetObject'],
      :resources => "arn:aws:s3:::#{bucket_name}/*",
      :principals => :any)
    bucket.policy = policy
  end
end

desc "Create the Amazon S3 billing Bucket"
task :billing_bucket do
  bucket_name = YAML.load_file(@config_filename)[:billing_bucket]
  begin
    bucket = @s3.buckets.create(bucket_name)
    puts "Billing Bucket! #{bucket_name}"
  end
  begin
    policy = AWS::S3::Policy.new
    policy.allow(
      :actions => ['s3:GetBucketAcl', 's3:GetBucketPolicy'],
      :resources => "arn:aws:s3:::#{bucket_name}",
      :principals => "arn:aws:iam::386209384616:root" )
    policy.allow(
      :actions => ['s3:PutObject'],
      :resources => "arn:aws:s3:::#{bucket_name}/*",
      :principals => "arn:aws:iam::386209384616:root" )
    bucket.policy = policy
    puts "Policy! #{bucket_name}"
    puts "Go here to enable Cost Allocation and Billing Reports:"
    puts "https://portal.aws.amazon.com/gp/aws/developer/account/index.html?ie=UTF8&ie=UTF8&action=billing-preferences"
    puts "Go here to manage Cost Allocation Reports:"
    puts "https://portal.aws.amazon.com/gp/aws/developer/account?ie=UTF8&action=cost-allocation-report"
  end
end

desc "Download Cost Allocation and Billing Reports"
task :reports do
  bucket_name = YAML.load_file(@config_filename)[:billing_bucket]
  reports_dir = YAML.load_file(@config_filename)[:reports_dir]
  bucket = @s3.buckets[bucket_name]
  objects = AWS::S3::ObjectCollection.new(bucket, :limit => 100)
  objects.each do |obj|
    path = Pathname.new("#{reports_dir}/#{obj.key}")
    if path.directory?
      path.dirname.mkpath
    else
      File.open(path, 'w') do |file|
        obj.read do |chunk|
          file.write(chunk)
        end
      end
    end
    puts path
  end
end

desc "Gather costs allocated to the CloudFormation stack from the cost allocation report."
task :cost do
  stack_name = YAML.load_file(@stack_filename)[:stack_name]
  reports_dir = YAML.load_file(@config_filename)[:reports_dir]
  files = FileList.new("#{reports_dir}/[0-9{12}]*aws-cost-allocation*.csv")
  puts "Actual! from:"
  puts files
  totals = Hash.new(0)
  files.each do |file|
    headers = File.open(file).readlines[1]
    CSV.foreach(file, :headers => headers, :converters => :numeric) do |row|
     if row['aws:cloudformation:stack-name'] == stack_name
       totals[row['aws:cloudformation:stack-name']] += row['CostBeforeTax']
     end
    end
  end
  puts totals
end

desc "Delete and then Create a CloudFormation stack."
task :replace => [:delete, :create]