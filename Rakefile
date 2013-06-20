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

@config = YAML.load_file(@config_filename)
@mappings_filename = @config[:mappings_filename]

@template_filename = 'template.json'
@parameters_filename = 'parameters.yml'
@outputs_filename = 'outputs.yml'
@stack_filename = 'stack.yml'
@log_filename = 'aws-sdk.log'
@userdata_filename_prefix = 'userdata_'


AWS.config(:access_key_id     => @config[:access_key_id], 
           :secret_access_key => @config[:secret_access_key],
           :logger => Logger.new(@log_filename),
           :log_formatter => AWS::Core::LogFormatter.colored)

@aws_regions = AWS.regions.collect

# for single region operations
@region = @config[:region] || 'us-east-1'

# for multi-region operations
@regions = @config[:regions].nil? ? regions = @aws_regions : regions = @config[:regions]

# AWS CloudFormation
@cfm = AWS::CloudFormation.new(:region => @region, :max_retries => 10)

# Amazon Simple Storage Service (S3) for staging
@s3 = AWS::S3.new(:region => @region)

# Amazon Simple Storage Service (S3) with assumed role for publishing
def s3_publish
  sts = AWS::STS.new
  user = sts.assume_role(:role_arn => @config[:publishing_role_arn], :role_session_name => "Upload")
  AWS::S3.new(user[:credentials])
end

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
  validation_result = @cfm.validate_template(File.open(@template_filename).read)
  if (validation_result[:message])
    puts validation_result[:message]
  else
    puts "CloudFormation Template Validated!"
  end
end

desc "Merge the Mapping and UserData sections."
task :merge => [:merge_mappings, :merge_userdata]

desc "Merge a Mappings section."
task :merge_mappings do
  mappings = JSON.parse(File.open(@mappings_filename).read)
  template = JSON.parse(File.open(@template_filename).read)
  template['Mappings'] = mappings
  template = JSON.pretty_generate(template)
  File.open(@template_filename, 'w') { |f| f.puts template }
  puts "Mappings merged into CloudFormation Template!"
end

desc "Merge resource specific Cloud-Init format UserData sections."
task :merge_userdata do
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
  File.open(@template_filename, 'w') { |f| f.puts template }
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
task :userdata do
  data = JSON.parse(File.open(@template_filename).read)['Resources']['Ec2Instance']['Properties']['UserData']['Fn::Base64']['Fn::Join'][1]
  File.open('userdata_Ec2Instance.sh', 'w') { |f| f.puts data.join }
  puts "UserData!"  
end

desc "Create a CloudFormation Stack."
task :create  => [:validate] do 
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
task :update => [:validate] do
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

desc "Publish the CloudFormation template and related assets to the staging bucket."
task :stage do
  upload @s3, @config[:staging_bucket]
end

desc "Publish the CloudFormation template and related assets to the production bucket."
task :publish do
  upload s3_publish, @config[:production_bucket]
end

def upload s3_client, bucket
  s3 = s3_client || @s3
  bucket_name   = bucket || @config[:staging_bucket]
  prefix        = @config[:staging_prefix]
  includes_file = @config[:includes_file]
  excludes_file = @config[:ignores_file]
  full_ctrl_acct_email = @config[:full_ctrl_acct_email]
  begin
    bucket = s3.buckets[bucket_name]
    includes = File.open(includes_file).read.split
    excludes = File.open(excludes_file).read.split
    file_names = FileList.new(includes).exclude(excludes)
    file_names.each do |file_name|
      path = Pathname.new(file_name)
      unless path.directory?
        key = "#{prefix}/#{path}"
        uri = "https://#{bucket_name}.s3.amazonaws.com/#{key}"
        bucket.objects[key].write(path)
        puts uri
      end
    end
  end
end

desc "Publish the CloudFormation template and related assets to the production bucket."
task :publish_manual do
  s3            = s3_publish
  timestamp     = DateTime.now.strftime("%s")
  bucket_name   = @config[:production_bucket]
  prefix        = @config[:staging_prefix]
  includes_file = @config[:includes_file]
  excludes_file = @config[:ignores_file]
  full_ctrl_acct_email = @config[:full_ctrl_acct_email]
  cloudfront_distribution = @config[:cloudfront_distribution]
  begin
    bucket = s3.buckets[bucket_name]
    includes = "*.pdf"
    excludes = File.open(excludes_file).read.split
    file_names = FileList.new(includes).exclude(excludes)
    file_names.each do |file_name|
      path = Pathname.new(file_name)
      unless path.directory?
        key = "#{prefix}/#{timestamp}/#{path}"
        bucket.objects[key].write(path)
        puts "https://#{cloudfront_distribution}/#{key}"
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

def register_keypair(keypair, public_key, region)
  public_key = Base64.encode64(public_key.to_der)
  ec2 = AWS::EC2.new :region => region
  begin
    ec2.key_pairs.import(keypair, public_key)
    puts "Imported EC2 KeyPair '#{keypair}' in AWS region #{region}."
  rescue AWS::EC2::Errors::InvalidKeyPair::Duplicate
    puts "EC2 KeyPair '#{keypair}' already exists in AWS region #{region}."
    return true
  end
end

def delete_keypair(keypair, region)
  ec2 = AWS::EC2.new :region => region
  begin
    ec2.key_pairs[keypair].delete
    puts "Deleted EC2 KeyPair '#{keypair}' in AWS region #{region}."
  rescue Error => e
    puts e
  end
end

task :keypair => [:keypair_delete , :keypair_create]

desc "Create an SSH EC2 KeyPair"
task :keypair_delete do
  keyname = @config[:keyname]
  @aws_regions.each { |region| delete_keypair(keyname, region.name) }
end

desc "Delete an SSH EC2 KeyPair"
task :keypair_create do
  keyname = @config[:keyname]
  # create an RSA keypair
  key = OpenSSL::PKey::RSA.new 2048
  # save the RSA keypair's private key
  File.open(".pk.pem", "w", 0600) do |f|
    f.write(key.to_pem)
  end
  # save the RSA keypair's public key
  File.open(".pub.pem", "w", 0600) do |f|
    f.write(key.public_key.to_pem)
  end
  public_key = key.public_key
  @aws_regions.each { |region| register_keypair(keyname, public_key, region.name) } 
end
