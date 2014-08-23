#!/usr/bin/env ruby

require "date"
require "cinch"
require "cinch/plugins/identify"
require "sinatra"
require "yaml"
require "json"
require "gitlab"

config_file = ARGV.shift || "config.yml"
if not File.exists? config_file
	puts "Can't find config file #{config_file}"
	puts "Either create it or specify another config file with: #{File.basename $0} [filename]"
	exit
end

$config = YAML.load_file config_file

$bot = Cinch::Bot.new do
	configure do |c|
		c.nick = $config["irc"]["nick"]
		c.user = "gitbot"
		c.realname = "GitBot"
		c.server = $config["irc"]["server"]
		c.port = $config["irc"]["port"]
		c.channels = $config["irc"]["channels"]
		c.plugins.plugins = [Cinch::Plugins::Identify]
		c.plugins.options[Cinch::Plugins::Identify] = {
			:password	=> $config["irc"]["password"],
			:type		=> :nickserv,
		}
	end
end

Gitlab.configure do |config|
	config.endpoint = $config["gitlab"]["endpoint"]
	config.private_token = $config["gitlab"]["token"]
end

Thread.new do
	$bot.start
end

def say(repo,msg)
	$config["irc"]["channels"].each do |chan|
		#unless $config["filters"].include? chan and not $config["filters"][chan].include? repo
			$bot.Channel(chan).send msg
		#end
	end
end

configure do
	set :bind, $config["http"]["host"]
	set :port, $config["http"]["port"]
	set :logging, false
	set :lock, true
end

post "/hook" do
	jsoncont = request.body.read
	p jsoncont
	push = JSON.parse(jsoncont)

	# new push
	if push["object_kind"] == nil
		repo = push["repository"]["name"]
		branch = push["ref"].gsub(/^refs\/heads\//,"")
		tag = push["ref"].gsub(/^refs\/tags\//,"")
		if branch != push["ref"]
			if push["commits"]
				say repo, "(new push) #{push["commits"][0]["message"]} (#{push["commits"][0]["id"].slice!(0..6)}) by \0033#{push["commits"][0]["author"]["name"]}\003 in \0037#{repo}\003 \[\0030#{branch}\003\]"
			else
				say repo, "(new branch) \[\0030#{branch}\003\] by \0033#{push["user_name"]}\003 in \0037#{repo}\003"
			end
		elsif tag != push["ref"]
			say repo, "(new tag) \[\0030#{tag}\003\] by \0033#{push["user_name"]}\003 in \0037#{repo}\003"
		end
	end

	# issue stuff
	if push["object_kind"] == "issue"
		if push["object_attributes"]["state"] == "opened"
			if push["object_attributes"]["created_at"] == push["object_attributes"]["updated_at"]
				pid = push["object_attributes"]["project_id"]
				repo = Gitlab.project(pid).name
				username = Gitlab.user(push["object_attributes"]["author_id"]).username
				issue_link = "http://#{$config["gitlab"]["host"]}/#{username}/#{repo}/issues/#{push["object_attributes"]["iid"]}"
				say repo, "(new issue) #{push["object_attributes"]["title"]} by \0033#{Gitlab.user(push["object_attributes"]["author_id"]).name}\003 [\##{push["object_attributes"]["iid"]} in \0037#{repo}\003] (#{issue_link})"
			else
				# something has been updated, let's investigate...
			end
		elsif push["object_attributes"]["state"] == "closed"
			pid = push["object_attributes"]["project_id"]
			repo = Gitlab.project(pid).name
			say repo, "(issue closed) \0030#{push["object_attributes"]["title"]}\003 [\##{push["object_attributes"]["iid"]} in \0037#{repo}\003]"
		elsif push["object_attributes"]["state"] == "reopened"

		end
	end

	# new merge request
	if push["object_kind"] == "merge_request"
		if push["object_attributes"]["state"] == "opened"

		elsif push["object_attributes"]["state"] == "closed"

		end
	end

	push.inspect
end
