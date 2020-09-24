# Magic's blog

## Setup

[Install dependencies](https://jekyllrb.com/docs/installation/ubuntu/):


```bash
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install ruby-full build-essential zlib1g-dev
echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

gem install jekyll bundler
```

## Starting the server

```bash
bundle install
bundle exec jekyll serve --livereload
```