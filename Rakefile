
require 'rubygems'
gem 'echoe', '>=2.6.4'
require 'echoe'

e = Echoe.new("mongrel") do |p|
  p.summary = "A small fast HTTP library and server that runs Rails, Camping, Nitro and Iowa apps."
  p.author ="Zed A. Shaw"
  p.clean_pattern = ['ext/http11/*.{bundle,so,o,obj,pdb,lib,def,exp}', 'ext/http11/Makefile', 'pkg', 'lib/*.bundle', '*.gem', 'site/output', '.config']
  p.rdoc_pattern = ['README', 'LICENSE', 'COPYING', 'lib/**/*.rb', 'doc/**/*.rdoc', 'ext/http11/http11.c']
  p.ignore_pattern = /^(pkg|site|projects|doc|log)|CVS|\.log/
  p.ruby_version = '>= 1.8.4'
  p.dependencies = ['gem_plugin >=0.2.3', 'cgi_multipart_eof_fix >=2.4']

  p.need_tar_gz = false
  p.need_tgz = true

  case RUBY_PLATFORM 
  when /mswin/
    p.certificate_chain = ['~/gem_certificates/mongrel-public_cert.pem', 
      '~/gem_certificates/luislavena-mongrel-public_cert.pem']
  else
    p.certificate_chain = ['~/p/configuration/gem_certificates/mongrel/mongrel-public_cert.pem', 
      '~/p/configuration/gem_certificates/evan_weaver-mongrel-public_cert.pem']
  end

  p.eval = proc do  
    case RUBY_PLATFORM
    when /mswin/
      extensions.clear
      self.files += ['lib/http11.so']
      self.platform = Gem::Platform::WIN32
    else
      add_dependency('daemons', '>= 1.0.3')
      add_dependency('fastthread', '>= 1.0.1')
    end
  end
  
end

#### Ragel builder

desc "Rebuild the Ragel sources"
task :ragel do
  Dir.chdir "ext/http11" do
    target = "http11_parser.c"
    File.unlink target if File.exist? target
    sh "ragel http11_parser.rl | rlgen-cd -G2 -o #{target}"
    raise "Failed to build C source" unless File.exist? target
  end
end

#### XXX Hack around RubyGems and Echoe for pre-compiled extensions.

def move_extensions
  Dir["ext/**/*.#{Config::CONFIG['DLEXT']}"].each { |file| cp file, "lib/" }
end

case RUBY_PLATFORM
when /mswin/
  filename = "lib/http11.so"
  file filename do
    Dir.chdir("ext/http11") do 
      ruby "extconf.rb"
      system(PLATFORM =~ /mswin/ ? 'nmake' : 'make')
    end
    move_extensions
  end 
  task :compile => [filename]
end

#### Project-wide install and uninstall tasks

def sub_project(project, *targets)
  targets.each do |target|
    Dir.chdir "projects/#{project}" do
      sh %{rake --trace #{target.to_s} }
    end
  end
end

desc "Package Mongrel and all subprojects"
task :package_all => [:package] do
  sub_project("gem_plugin", :package)
  sub_project("cgi_multipart_eof_fix", :package)
  sub_project("fastthread", :package)
  sub_project("mongrel_status", :package)
  sub_project("mongrel_upload_progress", :package)
  sub_project("mongrel_console", :package)
  sub_project("mongrel_cluster", :package)
  sub_project("mongrel_service", :package) if RUBY_PLATFORM =~ /mswin/
end

task :install_requirements do
  # These run before Mongrel is installed
  sub_project("gem_plugin", :install)
  sub_project("cgi_multipart_eof_fix", :install)
  sub_project("fastthread", :install)
end

desc "for Mongrel and all subprojects"
task :install => [:install_requirements] do
  # These run after Mongrel is installed
  sub_project("mongrel_status", :install)
  sub_project("mongrel_upload_progress", :install)
  sub_project("mongrel_console", :install)
  sub_project("mongrel_cluster", :install)  
  sub_project("mongrel_service", :install) if RUBY_PLATFORM =~ /mswin/
end

desc "for Mongrel and all its subprojects"
task :uninstall => [:clean] do
  sub_project("mongrel_status", :uninstall)
  sub_project("cgi_multipart_eof_fix", :uninstall)
  sub_project("mongrel_upload_progress", :uninstall)
  sub_project("mongrel_console", :uninstall)
  sub_project("gem_plugin", :uninstall)
  sub_project("fastthread", :uninstall)  
  sub_project("mongrel_service", :install) if RUBY_PLATFORM =~ /mswin/
end

desc "for Mongrel and all its subprojects"
task :clean do
  sub_project("gem_plugin", :clean)
  sub_project("cgi_multipart_eof_fix", :clean)
  sub_project("fastthread", :clean)
  sub_project("mongrel_status", :clean)
  sub_project("mongrel_upload_progress", :clean)
  sub_project("mongrel_console", :clean)
  sub_project("mongrel_cluster", :clean) 
  sub_project("mongrel_service", :clean) if RUBY_PLATFORM =~ /mswin/
end

#### Site upload tasks

namespace :site do

  desc "Package and upload .gem files and .tgz files for Mongrel and all subprojects to http://mongrel.rubyforge.org/releases/"
  task :source => [:package_all] do
    rm_rf "pkg/gems"
    rm_rf "pkg/tars"
    mkdir_p "pkg/gems"
    mkdir_p "pkg/tars"
   
    FileList["**/*.gem"].each { |gem| mv gem, "pkg/gems" }
    FileList["**/*.tgz"].each {|tgz| mv tgz, "pkg/tars" }
    
    # XXX Hack, because only Luis can package for Win32 right now
    sh "cp ~/Downloads/mongrel-1.0.2-mswin32.gem pkg/gems/"
    sh "cp ~/Downloads/mongrel_service-0.3.3-mswin32.gem pkg/gems/"  
    sh "rm -rf pkg/mongrel*"
    sh "gem generate_index -d pkg"  
    sh "scp -r CHANGELOG pkg/* rubyforge.org:/var/www/gforge-projects/mongrel/releases/" 
    sh "svn log -v > SVN_LOG"
    sh "scp -r SVN_LOG pkg/* rubyforge.org:/var/www/gforge-projects/mongrel/releases/" 
    rm "SVN_LOG"  
  end
  
  desc "Upload the website"
  task :web do
    # Requires the 'webgem' gem and the 'atom-tools' gem
    sh "cd site; webgen; ruby atom.rb > output/feed.atom; rsync -azv --no-perms --no-times output/* rubyforge.org:/var/www/gforge-projects/mongrel/"
  end
  
  desc "Upload the rdocs"
  task :rdoc => [:redoc] do
    sh "rsync -azv doc/* rubyforge.org:/var/www/gforge-projects/mongrel/rdoc/"
    sh "cd projects/gem_plugin; rake site"
  end
  
  desc "Upload the coverage report"
  task :coverage => [:rcov] do
    sh "rsync -azv test/coverage/* rubyforge.org:/var/www/gforge-projects/mongrel/coverage/"
  end
  
  desc "Upload the website, the rdocs, and the coverage report"
  task :all => [:web, :rdoc, :coverage]
  
end
