---
tags: snippet, perl, io
title: stdout autoflush
---

Классический метод:

	$oldfh = select(STDERR); $| = 1; select($oldfh);

ООП:

    use IO::Handle;
    STDERR->autoflush(1);