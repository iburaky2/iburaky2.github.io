# [iburaky.com](https://iburaky.com/)
My personal website based on [minimal](https://github.com/orderedlist/minimal).

## Local Setup
Install dependencies:
```
sudo apt install ruby-dev
sudo gem install bundler
bundle config set --local path vendor/bundle
bundle install
bundle exec jekyll build
```

Launch the website:
```
bundle exec jekyll serve --host 0.0.0.0 --port 4000 --livereload
```

## Generating Favicons
```
sudo apt install imagemagick

magick _src/favicon-source.png -filter Lanczos -resize 16x16   -strip favicon-16x16.png
magick _src/favicon-source.png -filter Lanczos -resize 32x32   -strip favicon-32x32.png
magick _src/favicon-source.png -filter Lanczos -resize 48x48   -strip favicon-48x48.png
magick _src/favicon-source.png -filter Lanczos -resize 180x180 -background '#ffffff' -alpha remove -alpha off -strip apple-touch-icon.png
magick _src/favicon-source.png -filter Lanczos -resize 192x192 -strip android-chrome-192x192.png
magick _src/favicon-source.png -filter Lanczos -resize 512x512 -strip android-chrome-512x512.png
magick _src/favicon-source.png -define icon:auto-resize=48,32,16 favicon.ico
```

## Generating Images
```
magick _src/posts/riscv-benchmark/lichee-rv-dock.png -strip -resize 1000x -quality 80 img/posts/riscv-benchmark/dock-1000.webp
magick _src/posts/riscv-benchmark/lichee-rv-dock.png -strip -resize 500x  -quality 80 img/posts/riscv-benchmark/dock-500.webp
```