# build
hugo --gc --minify
rm -rf ./public
rm -rf ~/code/website/website-deploy/lib/public
cp -r ./public ~/code/website/website-deploy/lib