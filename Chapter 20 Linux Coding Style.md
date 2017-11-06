## **前言**
这两天忙着应聘工作，博客更新有点慢。

## **Linux Coding Style**
It is not so important what style is chosen as long as one is indeed selected and used exclusively.

The majority of the style is covered in Linus's usual humor in the file *Documentation/CodingStyle* in the kernel source tree.

## **Indention**
Use tabs, but not eight spaces.

Linus rejoinder to this is that your code should not be so complex and convoluted as to require more than two or three levels of indention.

Need you go that deep, he argues, you should refactor you code to pull out layers of complexity into separate functions.

## **Switch Statements**
Subordinate *case* labels should be  indented to the same level as the parent *switch* statement.

**It is common practice to comment when deliberately falling through from one case statement to another**.

```c
switch(animal){
case ANIMAL_CAT:
	handle_cats();
	break;
case ANIMAL_WOLF:
	handle_wolves();
	/* fall through */
case ANIMAL_DOG:
	handle_dogs();
	break;
}
```

## **Spacing**
more keywords:
```c
if (foo)
while (foo)
for (i = 0; i < NR_CPUS; i++)
switch (foo)
```

functions, macros, keywords that look like functions:
```c
size_t nlongs = BITS_TO_LONG(nbits);
int len = sizeof(struct task_struct);
typeof(*p);
__alighof__(struct sockaddr *);
__attribute__((packed));
```

within parentheses, there is no space proceeding or preceding the argument, as previously shown.

most binary and tertiary operations:
```c
int sum = a + b;
int ret = (bar) ? bar : 0;
int nr = nr ? : 1; /* allow shortcut */
```

most unary operations:
```c
if (!foo)
foo++;
--bar;
```

Getting the spacing right around the dereference operator is particularly important:
```c
char *strcpy(char *dest, const char *src)
```

## **Braces**
most situations:
```c
do{
	percpu_counter_add(ca->cpustat[idx], val);
	ca = ca->parent;
}while (ca);
```

This rule is broken for functions, because functions:
```c
unsigned long func(void)
{
/* ... */
}
```

## **Line Length**
fewer than 80 characters in length;

```c
static void get_new_parrot(const char *name,
			   unsigned long disposition,
			   unsigned long feather_quality)
```

## **Naming**
global variables and functions should have descriptive names, in lowercase and delimited via an underscore as needed.
```c
get_active_tty()
```

## **Functions**
As rule of thumb, functions should not exceed one or two screens of text and should have fewer than ten local variables.

## **Comments**
describe what and why your code is doing what it is doing, not how it is doing it.

The how should be apparent from the code itself. If not, you might need to rethink and refactor what you wrote.

In comments, important notes are often prefixed with "XXX:", and bugs are often prefixed with "FIXME:", like so:
```c
/*
 * FIXME: we assume dog == cat which may not be true in the future
 */
```

special format:
```c
/**
 * find_treasure - find `X marks the spot`
 * @map - treasure map
 * @time - time the treasure was hidden
 *
 * Must call while holding the private_ship_lock.
 */
```

## **Typedef**
strong dislike *typedef*

## **Use Existing Routines**

## **Minimize ifdefs in the Source**
```c
#ifdef CONFIG_FOO
	foo()
#endif
```

Instead, define foo() to nothing if CONFIG_FOO is not set:
```c
#ifdef CONFIG_FOO
static int foo()
{
	/* ... */
}
#else
static inline int foo(void){}
#endif /* CONFIG_FOO */
```

## **Structure Initializers**
```c
struct foo my_foo{
	.a = INITIAL_A;
	.b = INITIAL_B;
};
```

## **Fixing Up Code Ex Post Facto**
If a pile of code falls into your lap that fails to even mildly resemble the Linux kernel coding style.
utility:
```
indent -kr -i8 -ts8 -sob -l80 -ss -bs -psl <file>
```
```
scripts/Lindent
```

## **Chain of Command**
