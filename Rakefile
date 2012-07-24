require 'rubygems'
require 'bundler'

Bundler.setup
Bundler.require


desc "Load the environment"
task :environment do
  env = ENV["SINATRA_ENV"] || "development"
  Sinatra::Base.environment = env
  Mongoid.load!("config/mongoid.yml")
  Mongoid.logger.level = Logger::INFO
  module CommentService
    class << self; attr_accessor :config; end
  end

  CommentService.config = YAML.load_file("config/application.yml")

  Dir[File.join(File.dirname(__FILE__),'models', '**', '*.rb')].each {|file| require file}

end

namespace :test do
  task :nested_comments => :environment do
    puts "checking"
    50.times do
      Comment.delete_all
      CommentThread.delete_all
      User.delete_all
      Notification.delete_all
      Subscription.delete_all
      
      user = User.create!(external_id: "1")

      comment_thread = CommentThread.new(title: "I can't solve this problem", body: "can anyone help me?", course_id: "1", commentable_id: "question_1")
      comment_thread.author = user
      comment_thread.save!

      comment = comment_thread.comments.new(body: "this problem is so easy", course_id: "1")
      comment.author = user
      comment.save!
      comment1 = comment.children.new(body: "not for me!", course_id: "1")
      comment1.author = user
      comment1.save!
      comment2 = comment1.children.new(body: "not for me neither!", course_id: "1")
      comment2.author = user
      comment2.save!

      children = comment_thread.comments.first.to_hash(recursive: true)["children"]
      if children.length == 2
        pp comment_thread.to_hash(recursive: true)
        pp comment_thread.comments.first.descendants_and_self.to_a
        puts "error!"
        break
      end
      puts "passed once"
    end
    puts "passed"
  end
end

task :console => :environment do
  binding.pry
end

namespace :db do
  task :init => :environment do
    puts "creating indexes..."
    Comment.create_indexes
    CommentThread.create_indexes
    User.create_indexes
    Notification.create_indexes
    Subscription.create_indexes
    Delayed::Backend::Mongoid::Job.create_indexes
    puts "finished"
  end

  task :clean => :environment do
    Comment.delete_all
    CommentThread.delete_all
    User.delete_all
    Notification.delete_all
    Subscription.delete_all
  end

  def generate_comments_for(commentable_id)
    level_limit = YAML.load_file("config/application.yml")["level_limit"]

    thread_seeds = [
      {title: "This is really interesting", body: "best I've ever seen!"},
      {title: "We can probably make this better", body: "Let's do it"},
      {title: "I don't know where to start", body: "Can anyone help me?"},
      {title: "I'm here!", body: "Haha I'm the first one who discovered this"},
      {title: "I need five threads but I don't know what to put here", body: "So I'll just leave it this way"},
    ]

    comment_body_seeds = [
      "dude I don't know what you're talking about",
      "hi I'm Jack",
      "hi just sent you a message",
      "let's discuss this further",
      "can't agree more",
      "haha",
      "lol",
    ]

    users = User.all.to_a

    puts "Generating threads and comments for #{commentable_id}..."

    threads = []
    comments = []

    thread_seeds.each do |thread_seed|
      comment_thread = CommentThread.new(commentable_id: commentable_id, body: thread_seed[:body], title: thread_seed[:title], course_id: "1")
      comment_thread.author = users.sample
      comment_thread.save!
      threads << comment_thread
      3.times do
        comment = comment_thread.comments.new(body: comment_body_seeds.sample, course_id: "1")
        comment.author = users.sample
        comment.endorsed = [true, false].sample
        comment.save!
        comments << comment
      end
      10.times do
        comment = Comment.where(comment_thread_id: comment_thread.id).reject{|c| c.depth >= level_limit}.sample
        sub_comment = comment.children.new(body: comment_body_seeds.sample, course_id: "1")
        sub_comment.author = users.sample
        sub_comment.endorsed = [true, false].sample
        sub_comment.save!
        comments << sub_comment
      end
    end

    puts "voting for these threads & comments.."

    threads.each do |c|
      users.each do |user|
        user.vote(c, [:up, :down].sample)
      end
    end

    comments.each do |c|
      users.each do |user|
        user.vote(c, [:up, :down].sample)
      end
    end

    puts "finished"
  end


  task :generate_comments, [:commentable_id] => :environment do |t, args|

    generate_comments_for(args[:commentable_id])

  end

  task :seed => :environment do

    Comment.delete_all
    CommentThread.delete_all
    User.delete_all
    Notification.delete_all
    Subscription.delete_all

    beginning_time = Time.now

    users = (1..10).map {|id| User.find_or_create_by(external_id: id.to_s)}

    10.times do
      users.sample.subscribe(users.sample)
    end
        
    generate_comments_for("question_1")
    generate_comments_for("question_2")
    generate_comments_for("course_1")
    generate_comments_for("lecture_1")
    generate_comments_for("video_1")
    generate_comments_for("video_2")
    generate_comments_for("lab_1")
    generate_comments_for("lab_2")

    end_time = Time.now

    puts "Number of comments generated: #{Comment.count}"
    puts "Number of comment threads generated: #{CommentThread.count}"

    puts "Time elapsed #{(end_time - beginning_time)*1000} milliseconds"

  end
end

# copied from https://github.com/sunspot/sunspot/blob/master/sunspot_solr/lib/sunspot/solr/tasks.rb
namespace :sunspot do
  namespace :solr do
    desc 'Start the Solr instance'
    task :start => :environment do
      case RUBY_PLATFORM
      when /w(in)?32$/, /java$/
        abort("This command is not supported on #{RUBY_PLATFORM}. " +
              "Use rake sunspot:solr:run to run Solr in the foreground.")
      end

      Sunspot::Solr::Server.new.start

      puts "Successfully started Solr ..."
    end

    desc 'Run the Solr instance in the foreground'
    task :run => :environment do
      Sunspot::Solr::Server.new.run
    end

    desc 'Stop the Solr instance'
    task :stop => :environment do
      case RUBY_PLATFORM
      when /w(in)?32$/, /java$/
        abort("This command is not supported on #{RUBY_PLATFORM}. " +
              "Use rake sunspot:solr:run to run Solr in the foreground.")
      end

      Sunspot::Solr::Server.new.stop

      puts "Successfully stopped Solr ..."
    end

  end

end
