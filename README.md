# build
rm -rf ./public
hugo --gc --minify
du -sh ./public
rm -rf ~/code/website/website-deploy/lib/public
cp -r ./public ~/code/website/website-deploy/lib