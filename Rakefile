require "rubygems"
require "bundler/setup"
require "stringex"

blog_index_dir  = 'source'    # directory for your blog's index page (if you put your index in source/blog/index.html, set this to 'source/blog')
stash_dir       = "_stash"    # directory to stash posts for speedy generation
posts_dir       = "_posts"    # directory for blog files
server_port     = "4000"      # port for preview server eg. localhost:4000

desc "Clean out caches and generated files"
task :clean do
  rm_rf [
    "deploy",
    "public",
    "source/stylesheets",
    ".gist-cache/**", 
    ".sass-cache/**", 
    ".pygments-cache/**", 
  ]
end

desc "Preview the site in a web browser"
task :develop do
  puts "Starting to watch source. Starting Rack on port #{server_port}"
  # jekyll's watch clobbers all files in the destination and wont watch files added to the source dir after its started, 
  # so compass must run first and output into jekyll's source dir
  compassPid = Process.spawn("compass watch --css-dir source/stylesheets --output-style nested")
  jekyllPid = Process.spawn({"OCTOPRESS_ENV"=>"preview"}, "jekyll --auto public")
  rackupPid = Process.spawn("rackup --port #{server_port}")

  trap("INT") {
    [jekyllPid, compassPid, rackupPid].each { |pid| Process.kill(9, pid) rescue Errno::ESRCH }
    exit 0
  }

  [jekyllPid, compassPid, rackupPid].each { |pid| Process.wait(pid) }
end

desc "Generate a release build of the site"
task :build do
  puts "\nGenerating Site with Jekyll"
  system "jekyll"
  puts "\nGenerating stylesheets with compass"
  system "compass compile --output-style compressed"
end

desc "Stage a release build in ./deploy to be committed to the gh-pages branch"
task :stage do
  Rake::Task[:clean].execute
  Rake::Task[:build].execute

  origin = `git config --get remote.origin.url`
  system "git fetch origin gh-pages:gh-pages"
  system "git clone -b gh-pages . deploy"
  cd 'deploy' do
    system "git config remote.origin.url #{origin}"
    system "git rm -rfq ."
    cp_r '../public/.', '.'
    system "git add ."
  end
end

desc "Commit a staged release in ./deploy and push to origin"
task :release do
  changes = `git status --porcelain`
  if changes.length > 0
    throw "ABORT: Can't release uncommitted changes"
  end

  # Rebuild the staged release to ensure it's current  
  Rake::Task[:stage].execute

  branch = `git symbolic-ref --short HEAD`
  rev = `git rev-parse HEAD`
  cd 'deploy' do
    message = "Released at #{Time.now.utc} from #{branch}@#{rev}"
    puts message
    system "git commit -m \"#{message}\""
    puts "\nPushing release to origin"
    system "git push origin gh-pages"
  end
end

# usage rake new_post[my-new-post] or rake new_post['my new post'] or rake new_post (defaults to "new-post")
desc "Begin a new post in source/#{posts_dir}"
task :new_post, :title do |t, args|
  mkdir_p "source/#{posts_dir}"
  args.with_defaults(:title => 'new-post')
  title = args.title
  filename = "source/#{posts_dir}/#{Time.now.strftime('%Y-%m-%d')}-#{title.to_url}.markdown"
  if File.exist?(filename)
    abort("rake aborted!") if ask("#{filename} already exists. Do you want to overwrite?", ['y', 'n']) == 'n'
  end
  puts "Creating new post: #{filename}"
  open(filename, 'w') do |post|
    post.puts "---"
    post.puts "layout: post"
    post.puts "title: \"#{title.gsub(/&/,'&amp;')}\""
    post.puts "date: #{Time.now.strftime('%Y-%m-%d %H:%M')}"
    post.puts "comments: true"
    post.puts "categories: "
    post.puts "---"
  end
end

# usage rake new_page[my-new-page] or rake new_page[my-new-page.html] or rake new_page (defaults to "new-page.markdown")
desc "Create a new page in source/(filename)/index.markdown"
task :new_page, :filename do |t, args|
  args.with_defaults(:filename => 'new-page')
  page_dir = ['source']
  if args.filename.downcase =~ /(^.+\/)?(.+)/
    filename, dot, extension = $2.rpartition('.').reject(&:empty?)         # Get filename and extension
    title = filename
    page_dir.concat($1.downcase.sub(/^\//, '').split('/')) unless $1.nil?  # Add path to page_dir Array
    if extension.nil?
      page_dir << filename
      filename = "index"
    end
    extension ||= 'markdown'
    page_dir = page_dir.map! { |d| d = d.to_url }.join('/')                # Sanitize path
    filename = filename.downcase.to_url

    mkdir_p page_dir
    file = "#{page_dir}/#{filename}.#{extension}"
    if File.exist?(file)
      abort("rake aborted!") if ask("#{file} already exists. Do you want to overwrite?", ['y', 'n']) == 'n'
    end
    puts "Creating new page: #{file}"
    open(file, 'w') do |page|
      page.puts "---"
      page.puts "layout: page"
      page.puts "title: \"#{title}\""
      page.puts "date: #{Time.now.strftime('%Y-%m-%d %H:%M')}"
      page.puts "comments: true"
      page.puts "footer: true"
      page.puts "---"
    end
  else
    puts "Syntax error: #{args.filename} contains unsupported characters"
  end
end

# usage rake isolate[my-post]
desc "Move all other posts than the one currently being worked on to a temporary stash location (stash) so regenerating the site happens much quicker."
task :isolate, :filename do |t, args|
  stash_dir = "source/#{stash_dir}"
  FileUtils.mkdir(stash_dir) unless File.exist?(stash_dir)
  Dir.glob("source/#{posts_dir}/*.*") do |post|
    FileUtils.mv post, stash_dir unless post.include?(args.filename)
  end
end

desc "Move all stashed posts back into the posts directory, ready for site generation."
task :integrate do
  FileUtils.mv Dir.glob("source/#{stash_dir}/*.*"), "source/#{posts_dir}/"
end

def get_stdin(message)
  print message
  STDIN.gets.chomp
end

def ask(message, valid_options)
  if valid_options
    answer = get_stdin("#{message} #{valid_options.to_s.gsub(/"/, '').gsub(/, /,'/')} ") while !valid_options.include?(answer)
  else
    answer = get_stdin(message)
  end
  answer
end
