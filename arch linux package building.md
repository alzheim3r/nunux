Arch linux Build Package
=======================

Get it 
------

    cd ~/src
    yaourt -G packagename 
    cd packagename
    # download but don't build
    makepkg -o

Customize it
------------

    nano PKGBUILD 


Build it 
--------

    # add -r to automatically remove build deps.
    makepkg -s


Install it 
----------

    pacman -U packagename.pkg.tar.xz 


Clean it 
--------

    rm -rf src pkg 

Can be useful 
-------------

Repackage without build.. (in case PKGBUILD need some post-build edit) 

    makepkg -R 

Sources 
-------

https://wiki.archlinux.org/index.php/Arch_Build_System 

