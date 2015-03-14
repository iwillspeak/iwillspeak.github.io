require 'yaml'
require 'rake/clean'

directory '_drafts'

desc "Create a new draft post"
task :new, [:title] => '_drafts' do |task, args|
  title = args[:title] || "New Post"
  mapped_title = title.strip.downcase.gsub(/\W+/, '-')
  header = {
    "title" => title,
    "layout" => "post",
    "published" => false
  }

  File.open("_drafts/#{mapped_title}.md", "w") do |f|
    f.write(YAML.dump(header))
    f.write("---\n")
  end
end
