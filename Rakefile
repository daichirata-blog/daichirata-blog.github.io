require 'rake'

desc 'Preview the site with Jekyll'
task :preview do
    sh "jekyll serve --host 0.0.0.0 --watch --drafts --config _config.yml,_config-dev.yml"
end

desc 'Search site and print specific deprecation warnings'
task :check do
    sh "jekyll doctor"
end
