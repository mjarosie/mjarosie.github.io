# Magic's blog

## Start the Jekyll server locally

```bash
docker compose up
```

## Manual setup

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

## With Docker

### Building the static page

```bash
export JEKYLL_VERSION=3.8.5
docker run --rm --volume="$PWD:/srv/jekyll" -it jekyll/jekyll:$JEKYLL_VERSION jekyll build
```

### Starting the server

```bash
export JEKYLL_VERSION=3.8.5
docker run --rm --volume="$PWD:/srv/jekyll" -p 4000:4000 -it jekyll/jekyll:$JEKYLL_VERSION jekyll serve --watch --drafts
```
