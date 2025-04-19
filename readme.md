# hljeong.com

## setup
```sh
git clone git@github.com:hljeong/hljeong.com.git
cd hljeong.com
make hooks
# todo: rest
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
```
