require 'pry'

# Include the lib files
libs=File.dirname(File.readlink(__FILE__))
Dir.glob("#{libs}/libs/*").each{|f| require f}

# coding: utf-8
task :default => :preview

# CONFIGURATION VARIABLES (on top of those defined by Jekyll in _config(_deploy).yml)
#
# PREVIEWING
# If your project is based on compass and you want compass to be invoked
# by the script, set the $compass variable to true
#
# $compass = false # default
# $compass = true  # if you wish to run compass as well
#
# Notice that Jekyll 2.0 supports sass natively, so you might want to have a look
# at the functions provided by Jekyll instead of using the functions provided here.
#
# MANAGING POSTS
# Set the extension for new posts (defaults to .textile, if not set)
#
# $post_ext = ".textile"  # default
# $post_ext = ".md"       # if you prefer markdown
#
# Set the location of new posts (defaults to "_posts/", if not set).
# Please, terminate with a slash:
#
# $post_dir = "_posts/"
#
# MANAGING MULTI-USER WORK
# If you are using git to manage the sources, you might want to check the repository
# is up-to-date with the remote branch, before deploying.  In fact---when this is not the
# case---you end up deploying a previous version of your website.
#
# The following variable determines whether you want to check the git repository is
# up-to-date with the remote branch and, if not, issue a warning.
#
# $git_check = true
#
# It is safe to leave the variable set to true, even if you do not manage your sources
# with git.
#
# The following variable controls whether we push to the remote branch after deployment,
# committing all uncommitted changes
#
# $git_autopush = false
#
# If set to true, the sources have to be managed by git or an error message will be issued.
#
# ... or load them from the configuration file, e.g.:
# 
load '_rake-configuration.rb' if File.exist?('_rake-configuration.rb')
load '_rake_configuration.rb' if File.exist?('_rake_configuration.rb')

# Specify default values for variables NOT set by the user

$post_ext ||= ".markdown"
$post_dir ||= "_posts/"
$git_check ||= true
$git_autopush ||= false

#
# Tasks start here
#

desc 'Clean up generated site'
task :clean do
  cleanup
end


desc 'Preview on local machine (server with --auto)'
task :preview => :clean do
  compass('compile') # so that we are sure sass has been compiled before we run the server
  compass('watch &')
  jekyll('serve --watch')
end
task :serve => :preview


desc 'Build for deployment (but do not deploy)'
task :build, [:deployment_configuration] => :clean do |t, args|
  args.with_defaults(:deployment_configuration => 'deploy')
  config_file = "_config_#{args[:deployment_configuration]}.yml"

  if rake_running then
    puts "\n\nWarning! An instance of rake seems to be running (it might not be *this* Rakefile, however).\n"
    puts "Building while running other tasks (e.g., preview), might create a website with broken links.\n\n"
    puts "Are you sure you want to continue? [Y|n]"

    ans = STDIN.gets.chomp
    exit if ans != 'Y' 
  end

  compass('compile')
  jekyll("build --config _config.yml,#{config_file}")
end

desc 'Create a draft post'
task :new_draft, [:title, :content] do |t, args|
  # Todo:
  # - Check we are on the drafts branch
  # - If not move to drafts branch
  # - Error if cannot move (ie non-checked in changes)
  $post_dir = "_drafts/"
  create_new_post(t, args)
end

desc 'Build and deploy to remote server'
task :deploy, [:deployment_configuration] => :build do |t, args|
  args.with_defaults(:deployment_configuration => 'deploy')
  config_file = "_config_#{args[:deployment_configuration]}.yml"

  text = File.read("_config_#{args[:deployment_configuration]}.yml")
  matchdata = text.match(/^deploy_dir: (.*)$/)
  if matchdata

    if git_requires_attention("master") then
      puts "\n\nWarning! It seems that the local repository is not in sync with the remote.\n"
      puts "This could be ok if the local version is more recent than the remote repository.\n"
      puts "Deploying before committing might cause a regression of the website (at this or the next deploy).\n\n"
      puts "Are you sure you want to continue? [Y|n]"

      ans = STDIN.gets.chomp
      exit if ans != 'Y' 
    end

    deploy_dir = matchdata[1]
    sh "rsync -avz --delete _site/ #{deploy_dir}"
    time = Time.new
    File.open("_last_deploy.txt", 'w') {|f| f.write(time) }
    %x{git add -A && git commit -m "autopush by Rakefile at #{time}" && git push} if $git_autopush
  else
    puts "Error! deploy_url not found in _config_deploy.yml"
    exit 1
  end
end

desc 'Build and deploy to github'
task :deploy_github => :build do |t, args|
  args.with_defaults(:deployment_configuration => 'deploy')
  config_file = "_config_#{args[:deployment_configuration]}.yml"

  if git_requires_attention("gh_pages") then
    puts "\n\nWarning! It seems that the local repository is not in sync with the remote.\n"
    puts "This could be ok if the local version is more recent than the remote repository.\n"
    puts "Deploying before committing might cause a regression of the website (at this or the next deploy).\n\n"
    puts "Are you sure you want to continue? [Y|n]"

    ans = STDIN.gets.chomp
    exit if ans != 'Y' 
  end

  %x{git add -A && git commit -m "autopush by Rakefile at #{time}" && git push origin gh_pages} if $git_autopush
  
  time = Time.new
  File.open("_last_deploy.txt", 'w') {|f| f.write(time) }
end

desc 'Create a post listing all changes since last deploy'
task :post_changes do |t, args|
  content = list_file_changed
  Rake::Task["create_post"].invoke(Time.new.strftime("%Y-%m-%d %H:%M:%S"), "Recent Changes", nil, content)
end


desc 'Show the file changed since last deploy to stdout'
task :list_changes do |t, args|
  content = list_file_changed
  puts content
end


#
# support functions for generating list of changed files
#

def list_file_changed
  content = "Files changed since last deploy:\n"
  IO.popen('find * -newer _last_deploy.txt -type f') do |io| 
    while (line = io.gets) do
      filename = line.chomp
      if user_visible(filename) then
        content << "* \"#{filename}\":{{site.url}}/#{file_change_ext(filename, ".html")}\n"
      end
    end
  end 
  content
end

# this is the list of files we do not want to show in changed files
EXCLUSION_LIST = [/.*~/, /_.*/, "javascripts?", "js", /stylesheets?/, "css", "Rakefile", "Gemfile", /s[ca]ss/, /.*\.css/, /.*.js/, "bower_components", "config.rb"]

# return true if filename is "visible" to the user (e.g., it is not javascript, css, ...)
def user_visible(filename)
  exclusion_list = Regexp.union(EXCLUSION_LIST)
  not filename.match(exclusion_list)
end 

def file_change_ext(filename, newext)
  if File.extname(filename) == ".textile" or File.extname(filename) == ".md" then
    filename.sub(File.extname(filename), newext)
  else  
    filename
  end
end


desc 'Check links for site already running on localhost:4000'
task :check_links do
  begin
    require 'anemone'

    root = 'http://localhost:4000/'
    puts "Checking links with anemone ... "
    # check-links --no-warnings http://localhost:4000
    Anemone.crawl(root, :discard_page_bodies => true) do |anemone|
      anemone.after_crawl do |pagestore|
        broken_links = Hash.new { |h, k| h[k] = [] }
        pagestore.each_value do |page|
          if page.code != 200
            referrers = pagestore.pages_linking_to(page.url)
            referrers.each do |referrer|
              broken_links[referrer] << page
            end
          else
            puts "OK #{page.url}"
          end
        end
        puts "\n\nLinks with issues: "
        broken_links.each do |referrer, pages|
          puts "#{referrer.url} contains the following broken links:"
          pages.each do |page|
            puts "  HTTP #{page.code} #{page.url}"
          end
        end
      end
    end
    puts "... done!"

  rescue LoadError
    abort 'Install anemone gem: gem install anemone'
  end
end


#
# General support functions
#

# remove generated site
def cleanup
  sh 'rm -rf _site'
  compass('clean')
end

# launch jekyll
def jekyll(directives = '')
  sh 'jekyll ' + directives
end

# launch compass
def compass(command = 'compile')
  (sh 'compass ' + command) if $compass
end

# check if there is another rake task running (in addition to this one!)
def rake_running
  `ps | grep 'rake' | grep -v 'grep' | wc -l`.to_i > 1
end

def git_local_diffs
  %x{git diff --name-only} != ""
end

def git_remote_diffs branch
  %x{git fetch}
  %x{git rev-parse #{branch}} != %x{git rev-parse origin/#{branch}}
end

def git_repo?
  %x{git status} != ""
end

def git_requires_attention branch
  $git_check and git_repo? and git_remote_diffs(branch)
end
