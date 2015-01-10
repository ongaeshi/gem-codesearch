require 'rbconfig'
require 'fileutils'
require 'find'
require 'pp'

task :default => :help

task :help do
  puts <<'End'
Usage:
  rake help
  rake all              # mirror, unpack, index
  rake mirror
  rake unpack
  rake index
End
end

task :all => [:mirror, :unpack, :index]

MIRROR_DIR = "#{Dir.pwd}/mirror"
LATEST_DIR = "#{Dir.pwd}/latest-gem"
LOG_DIR = "#{Dir.pwd}/log"

GEM_COMMAND = "#{RbConfig::CONFIG["bindir"]}/gem"
MILK_COMMAND = "#{RbConfig::CONFIG["bindir"]}/milk"

file ".gem/.mirrorrc" do |t|
  FileUtils.mkpath File.dirname(t.name)
  File.write(t.name, <<"End")
---
- from: http://rubygems.org/
  to: #{MIRROR_DIR}
End
end

task :mirror => ".gem/.mirrorrc" do
  FileUtils.mkpath MIRROR_DIR
  # HOME is set because gem mirror reads $HOME/.gem/.mirrorrc.
  # sh "HOME=#{Dir.pwd} #{GEM_COMMAND} mirror --verbose"

  current_dir = Dir.pwd
  $:.unshift File.join(current_dir, '../rubygems-mirror/lib')
  require 'rubygems/mirror/command'

  mirror = Gem::Commands::MirrorCommand.new
  mirror.execute(current_dir)
end

task :unpack do
  FileUtils.mkpath LOG_DIR

  available_gems = {}
  Dir.foreach("#{MIRROR_DIR}/gems") {|filename|
    next if /\.gem\z/ !~ filename
    available_gems[filename] = true
  }
  FileUtils.mkpath LATEST_DIR
  all_specs = File.open("#{MIRROR_DIR}/specs.4.8") {|f| Marshal.load(f) }
  all_specs = all_specs.reject {|name,version,platform|
    /\A\./ =~ name ||
    /\A[0-9a-zA-Z._-]+\z/ !~ name ||
    /\A[0-9a-zA-Z._-]+\z/ !~ version.to_s ||
    platform != 'ruby' ||
    !available_gems["#{name}-#{version}.gem"]
  }

  #all_specs = all_specs.reject {|name,version,platform| /\Afoo-/ !~ name }

  latest_vnames = []
  nonlatest_vnames = []
  h = all_specs.group_by {|name,version| name }
  h.each {|name, list|
    list = list.sort_by {|name,version| version }
    vnames = list.map {|name,version| "#{name}-#{version}" }
    latest_vnames << vnames.pop
    nonlatest_vnames.concat list
  }

  already_unpacked = Dir.entries(LATEST_DIR)
  (already_unpacked & nonlatest_vnames).each {|vname|
    puts "remove: #{vname}
    FileUtils.rmtree("#{LATEST_DIR}/#{vname}")
  }

  File.open("#{LOG_DIR}/unpack.log.#{Time.now.strftime '%FT%T%:z'}", "a") {|log|
    (latest_vnames - already_unpacked).each {|vname|
      puts "unpack: #{vname}"
      system GEM_COMMAND, 'unpack', "#{MIRROR_DIR}/gems/#{vname}.gem", :chdir => LATEST_DIR, :out => log
      if !$?.success?
        puts "failed to unpack #{vname}"
      end
      fix_permission("#{LATEST_DIR}/#{vname}")
    }
  }

end

task :index do
  # Assume default database for milkode is already created.
  # If not, do it as follows:
  #   milk init --default
  milkode_package_list = IO.popen([MILK_COMMAND, 'list']) {|f| f.read }
  package_name = File.basename(LATEST_DIR)
  if /^#{Regexp.escape package_name}$/ !~ milkode_package_list
    system MILK_COMMAND, 'add', '--verbose', LATEST_DIR
  else
    system MILK_COMMAND, 'update', '--verbose', package_name
  end
end

def fix_permission(dir)
  return unless File.exist? dir
  Find.find(dir) {|fn|
    st = File.lstat(fn)
    if st.file?
      if !st.readable?
        File.chmod(0644, fn)
      end
    elsif st.directory?
      if !st.readable? || !st.executable?
        File.chmod(0755, fn)
      end
    end
  }
end
