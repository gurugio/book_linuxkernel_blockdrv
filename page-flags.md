# page-flags

We have seen page flags many times.
Each page has different page flags.

Page flags are defined in include/linux/page-flags.h file.
Basically every flags are defined as enum values in enum pageflags, for instance PG_reserved and PG_private.
And you can find description for each flag.

But flag handling functions, for instance `__SetPageReferenced`, are not there.
You cannot find them with grep tool and any tagging tools.
That is because they are defined with macro in page-flags.h

For example, following is definition of PG_locked.

```
TESTPAGEFLAG(Locked, locked)
```

What is TESTPAGEFLAG macro?
```
#define TESTPAGEFLAG(uname, lname)    				\
static inline int Page##uname(const struct page *page)			\
			{ return test_bit(PG_##lname, &page->flags); }
```

uname is Locked and lname is locked.
Let's translate the macro into real code.

```
static inline int PageLocked(const struct page *page)
			{ return test_bit(PG_locked, &page->flags); }
```

Finally we can understand that TESTPAGEFLAG macro defines a bit checking function PageLocked.
`PG_` prefix is omiited to define macro.

Let's check another macro using lru flag.

```
PAGEFLAG(LRU, lru) __CLEARPAGEFLAG(LRU, lru)
```

This also omits `PG_` prefix of flag name PG_lru.
Let's check what PAGEFLAG and CLEARPAGEFLAG are.

```
#define PAGEFLAG(uname, lname) TESTPAGEFLAG(uname, lname)    	\
	SETPAGEFLAG(uname, lname) CLEARPAGEFLAG(uname, lname)

#define TESTPAGEFLAG(uname, lname)    				\
static inline int Page##uname(const struct page *page)			\
			{ return test_bit(PG_##lname, &page->flags); }

#define SETPAGEFLAG(uname, lname)    				\
static inline void SetPage##uname(struct page *page)			\
			{ set_bit(PG_##lname, &page->flags); }

#define CLEARPAGEFLAG(uname, lname)    				\
static inline void ClearPage##uname(struct page *page)			\
			{ clear_bit(PG_##lname, &page->flags); }

#define __CLEARPAGEFLAG(uname, lname)					\
static inline void __ClearPage##uname(struct page *page)		\
			{ __clear_bit(PG_##lname, &page->flags); }
```

Finally PAGEFLAG and `__CLEARPAGEFLAG` macros generates PageLru, SetPageLru, ClearPageLru, `__ClearPageLru` macro functions.

We can find out a nameing policy here.
* Page<Flag>: check flag
* SetPage<Flag>: set flag
* ClearPage<Flag>: clear flag
* `__ClearPage<Flag>: clear flag without lock

Now we can understand what each macro sets/clears which flag.
