---
layout: post
title:  "Markdown and Jekyll"
date:   2017-07-29 22:14:34 +0800
categories: markdown jekyll
---
在GitHub專案中, 常常可以看到README.md檔, 這個檔案就是專案首頁呈現的內容. md是Markdown的縮寫, 是John Gruber發明的文件格式, 其用意是利用純文字產生XHTML/HTML文件, 並保持純文字的可讀性. Markdown的語法簡單易學, 可以參考[Markdown Project][markdown]的內容學習.

[markdown]: https://daringfireball.net/projects/markdown/

[Jekyll][]是一種static website/blog的排版引擎, 使用者可以撰寫markdown的純文字內容, 透過Jekyll輸出整個網站的內容, 不用費心版面的設計, 專注在內容的寫作. Jekyll也是GitHub Pages的預設排版工具, 所以就去開了個帳號, 用來練習一下Markdown語法, 順便養成把study過的東西紀錄起來的習慣. Jekyll的安裝很容易, 在我的Ubuntu 14.04上, 步驟如下

1.  Jekyll需要ruby 2.1以上, 但是Ubuntu官方沒有符合的版本, 需要新增third-party的ppa
```bash
sudo apt-add-repository ppa:brightbox/ruby-ng
sudo apt-get update
```
2.  安裝ruby 2.3
```bash
sudo apt-get install ruby2.3
sudo apt-get install ruby2.3-dev
```
3.  利用gem安裝jekyll, bundler
```bash
sudo gem install jekyll bundler
```
4.  產生blog, 並在本機browser觀看輸出結果, http://localhost:4000/
```bash
jekyll new myblog
cd myblog
bundle exec jekyll serve
```

上傳到GitHub也很容易, 整個myblog目錄的內容往repository丟, 不該上傳的部份, jekyll已經自動產生.gitignore幫你做好防呆了.

[jekyll]: https://jekyllrb.com/
