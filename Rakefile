require "date"
require "shellwords"

def shellescape(value)
  Shellwords.escape(value.to_s)
end

desc "Build the Docker image"
task :install do
  sh "docker compose build"
end

desc "Run the Jekyll blog in Docker"
task :serve do
  sh "docker compose up --build"
end

desc "Stop the Docker services"
task :stop do
  sh "docker compose down"
end

namespace :docker do
  desc "Build the Docker image"
  task :build do
    sh "docker compose build"
  end

  desc "Build and run the Jekyll blog in Docker"
  task :serve do
    sh "docker compose up --build"
  end

  desc "Stop Docker services"
  task :stop do
    sh "docker compose down"
  end

  desc "Open a shell inside the Jekyll container"
  task :shell do
    sh "docker compose run --rm jekyll bash"
  end
end

desc "Create a new post: rake post title='My Post Title'"
task :post do
  title = ENV["title"].to_s.strip

  abort("Usage: rake post title='My Post Title'") if title.empty?

  slug = title.downcase.gsub(/[^a-z0-9]+/, "-").gsub(/^-|-$/, "")
  date = Date.today.strftime("%Y-%m-%d")
  path = File.join("_posts", "#{date}-#{slug}.md")

  sh <<~CMD.gsub(/\s+/, " ").strip
    docker compose run --rm
    -e POST_TITLE=#{shellescape(title)}
    -e POST_PATH=#{shellescape(path)}
    jekyll ruby -e "
      path = ENV.fetch('POST_PATH');
      title = ENV.fetch('POST_TITLE');
      abort(%(Post already exists: \#{path})) if File.exist?(path);
      File.write(path, <<~POST)
      ---
      title: \#{title}
      description:
      ---

      Write your post here.
      POST
      puts %(Created \#{path})
    "
  CMD
end
