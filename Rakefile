require 'yaml'
require 'rake/clean'

DRAFTS_DIR = '_drafts'
POSTS_DIR = '_posts'
SITE_DIR = '_site'

directory "#{DRAFTS_DIR}"

class String
  def slugify()
    self.strip.downcase.gsub(/\W+/, '-')
  end
end

def write_post(f, header, body)
  f.write(YAML.dump(header))
  f.write('---')
  f.write(body)
end

def most_recent_draft()
  (Dir.glob("#{DRAFTS_DIR}/*.md").sort_by { |f| File.mtime f }).last
end

def find_draft_from_hint(hint)
  return most_recent_draft if hint.nil?

  if File.exists? hint
    hint
  else
    Dir.glob("#{DRAFTS_DIR}/*#{hint}*.md").first
  end
end

desc "Create a new draft post"
task :new, [:title] => "#{DRAFTS_DIR}" do |task, args|
  title = args[:title] || Date.today.strftime("Draft Post %F")
  mapped_title = title.slugify()
  header = {
    "title" => title,
    "layout" => "post"
  }

  File.open("#{DRAFTS_DIR}/#{mapped_title}.md", "w") do |f|
    write_post(f, header, "")
  end
end

desc "Edit a Draft"
task :edit, [:globs] do |task, args|
  file = find_draft_from_hint(args[:globs])
  spawn %{writer "#{file}"}
end

desc "Publish a draft"
task :publish, [:globs] do |task, args|
  file = find_draft_from_hint(args[:globs])
  
  draft = File.open(file, "r").read

  header = YAML.load(draft)
  body = draft.split("---", 3)[2]

  header["published"] = true

  post_name = header["title"] || File.basename(file, ".*")
  post_name = post_name.slugify()
  post_name = Date.today.strftime("%F-#{post_name}")

  File.open("#{POSTS_DIR}/#{post_name}.md", "w") do |f|
    write_post(f, header, body)
  end
end

desc "Serve the site with jekyll"
task :serve do
  sh %{jekyll serve --watch --drafts}
end

desc "Build the site"
task :build do
  sh %{jekyll build}
end

task :default => :build

CLEAN << SITE_DIR
