Archiver commands cheatsheet
============================

## 7zip

compress:

    7z u -mx=9 archive.7z sourcedir
    tar cpf - sourcedir | 7z a -ms=on -mx=9 -si archive.tar.7z

extract:

    7z x archive.7z
    7z x -so archive.tar.7z | tar xvf -

## xz

multithread + maximum compression

    export XZ_OPT="--threads=4 -e9" tar -cJf archive.tar.xz /folder/
