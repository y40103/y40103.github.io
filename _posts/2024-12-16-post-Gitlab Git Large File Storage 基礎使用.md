---
title: "[筆記] Gitlab Git Large File Storage 基礎使用"
categories:
  - 筆記
tags:
  - Git
toc: true
toc_label: Index
mermaid: true
---


可以將一些binary file push至git上, 這些檔案較大, 且不希望去追蹤文檔變更   

git lfs作法是將該檔案的指標存在repository上, 實際檔案則是另外存放  


## Install

```
sudo apt-get install git-lfs
sudo apt install git-filter-repo
```


## add lfs

target: MWPAlqP.zip

```bash
git lfs track MWPAlqP.zip
# 會把 MWPAlqP.zip 加到 .gitattributes 這個檔案

git add .
# 把 MWPAlqP.zip 與 .gitattributes 加入

git commit -m 'add lfs'

git push
```


## purge lfs

由於git 是versioning特性, 若要真的移除檔案, 需處理所有的歷史commit

```bash
git filter-repo --force --path MWPAlqP.zip --invert-paths
git add .
git commit -m 'purge remote lfs'
git push --force --all `repo-url`
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

手動刪除

Settings > General > advanced > Prune unreachable objects


## Reference


[Deleted the LFS objects from GitLab repository however storage space still not freed up](https://stackoverflow.com/questions/77110174/deleted-the-lfs-objects-from-gitlab-repository-however-storage-space-still-not-f)
