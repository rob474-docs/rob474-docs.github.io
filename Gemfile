source 'https://rubygems.org'

gem "jekyll", "~> 4.3"
gem "webrick"   # required by `jekyll serve` on Ruby >= 3

# If you have any plugins, put them here!
group :jekyll_plugins do
  gem "jekyll-remote-theme"
  gem "jekyll-last-modified-at"

  # Runtime dependencies of the just-the-docs theme. Because the theme is
  # pulled in via remote_theme rather than as a gem, bundler can't resolve
  # these for us -- they have to be declared here.
  gem "jekyll-seo-tag"
  gem "jekyll-include-cache"
end
