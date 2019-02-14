task :default => :new

require 'fileutils'

desc "新增post"
task :new do
  puts "請輸入要新增的 post URL："
	@url = STDIN.gets.chomp
	puts "請輸入 post 標題："
	@name = STDIN.gets.chomp
	puts "請輸入 post 分類，以空格分隔："
	@categories = STDIN.gets.chomp
	@slug = "#{@url}"
	@slug = @slug.downcase.strip.gsub(' ', '-')
	@date = Time.now.strftime("%F")
	@post_name = "_posts/#{@date}-#{@slug}.md"
	@author = "Double Chuang"
	if File.exist?(@post_name)
			abort("文件名已經存在！創建失敗")
	end
	FileUtils.touch(@post_name)
	open(@post_name, 'a') do |file|
			file.puts "---"
			file.puts "layout:     post"
			file.puts "title:      \"#{@name}\""
			file.puts "date:       #{Time.now}"
			file.puts "author:     \"#{@author}\""
			file.puts "categories: #{@categories}"
			file.puts "---"
	end
	exec "vim #{@post_name}"
end
