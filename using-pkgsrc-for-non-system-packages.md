Using pkgsrc for system packages
================================

I chose Debian in order to keep my local development environment better aligned with GitHub's operating environment.
But my attempts to fill in the gaps in available software by creating my own Debian packages quickly became more frustrating than I was willing to invest in.
So I've turned to pkgsrc to provide some more up-to-date and customizable versions of packages that I use.

I don't know pkgsrc's format as well as I do Arch's but it sure as shit can't be worse than Debian's.

## Setup

I forgot to take setup notes, but I referenced https://imil.net/blog/2015/07/05/using-pkgsrc-on-debian-gnulinux/
		# apt-get install cvs libncurses5-dev gcc g++ zlib1g-dev zlib1g libssl-dev libudev-dev
		# cd /usr; and cvs -d anoncvs@anoncvs3.de.NetBSD.org:/cvsroot co pkgsrc
		# cd pkgsrc
		# env SH=/bin/bash ./bootstrap/bootstrap --prefix=/usr/local

`/usr/local/sbin` and `/usr/local/bin` are already part of root's path but not my user's.

		> set -g fish_user_paths /usr/local/bin $fish_user_paths

## Package configuration

So far the only package I've needed to customize is vim.

		# cd /usr/pkgsrc/editors/vim
		# bmake show-options
		# echo "PKG_OPTIONS.vim= lua luajit perl python ruby" >> /usr/local/etc/mk.conf
		# bmake install



