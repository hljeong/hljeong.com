# hljeong.com

## setup
```sh
git clone git@github.com:hljeong/hljeong.com.git
cd hljeong.com
make hooks
# todo: rest
#       ^ 3/22/26: tf do u mean by rest
```

## add post
```sh
hugo new content content/posts/<title>.md
```

## start dev server
```sh
hugo server -D
```

## deploy
```sh
hugo && firebase deploy

# if case of "resolving hosting target of a site with no site name or target name":
firebase login --reauth
```
