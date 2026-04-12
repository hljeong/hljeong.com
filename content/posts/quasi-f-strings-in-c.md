---
title: "we have f-strings at home: quasi-f-strings in c"
date: 2026-04-11T21:34:32-07:00
tags: ["computer science"]
---

afraid not, here is a quick rant completely unrelated to the rest of the content:
overhyping shit is so ass. its much better to have low expectations, or better yet, not know what to expect. gone are the days when steve jobs could change the world with a single presentation announcing the iphone. now theyve overedged my ass so much that when agi befalls us and ruins the world, ill be fatigued and unimpressed

anyways, back to the regularly scheduled programming. ive been taking a liking to writing in c maybe because of how it forces you to reinvent wheels, which i love to do. i ran into 2 issues:
1. i have a function to print the ast to `stderr`. its recursive and prints line-by-line. a need to print the ast to `stdout` emerges, and i need to copy the entire function and replace occurrences of `debugf()`, which is an alias for `fprintf(stderr)`, with `printf()`
   ```c
   void debug_ast(const Node *node) {
     debugf("something %s\n", node->something);
     const Node *child = node->child;
     while (child) {
       debug_ast(child);
       child = child->next;
     }
   }

   void print_ast(const Node *node) {
     printf("something %s\n", node->something);
     const Node *child = node->child;
     while (child) {
       print_ast(child);
       child = child->next;
     }
   }
   ```
2. `to_str` functions are implemented for objects such as tokens so that they can be interpolated into debug logs. this is fine for tokens with static names, _e.g._, the string representation of a semicolon token is just `";"`. this breaks down when the string representation is not static: the string representation of an identifier has to contain the lexeme, which poses issues with dynamic allocation, ownership, lifetime, and memory leaks
   ```c
   const char *token_to_str(const Token *tok) {
     char buf[BUF_SIZE];
     switch (tok->kind) {
       case TokenKind_SEMICOLON: return ";";
       case TokenKind_IDENTIFIER: {
         snprintf(buf, sizeof(buf), "ident(%s)", tok->lexeme);

         // option 1:
         return buf;
         // ^ lifetime issue--what if caller runs:
         //   printf("%s %s\n", token_to_str(ident1),
         //                     token_to_str(ident2));
         //
         //   prints ident2 twice

         // option 2:
         return strdup(buf);
         // ^ non-owning, caller has to:
         //   const char *ident1_str = token_to_str(ident1);
         //   const char *ident2_str = token_to_str(ident2);
         //   printf("%s %s\n", ident1_str, ident2_str);
         //   free(ident1_str);
         //   free(ident2_str);
         //
         //   might as well rename the function to
         //   token_to_str_and_by_the_way_fuck_you()
       }
       default: return "unknown";
     }
   }
   ```

the first problem is a perfect use case for the "strategy pattern" (god i fucking hate the so-called design patterns but alas it gets the point across q precisely here), to enable selecting an arbitrary "consequence" for the string generated. lets take an `int (*emit)(const char *fmt, ...)` as argument (shoutouts to the god awful function pointer syntax in c):
```c
void emit_ast(int (*emit)(const char* fmt, ...), const Node *node) {
  emit("something %s\n", node->something);
  const Node *child = node->child;
  while (child) {
    emit_ast(emit, child);
    child = child->next;
  }
}

void debug_ast(const Node *node) { emit_ast(debugf, node); }

void print_ast(const Node *node) { emit_ast(printf, node); }
```

this actually helps solve the second issue too, since it eagerly consumes the produced string fragment:
```c
void emit_token(int (*emit)(const char *fmt, ...), const Token *tok) {
  switch (tok->kind) {
    case TokenKind_SEMICOLON:  { emit(";"); break; }
    case TokenKind_IDENTIFIER: { emit("ident(%s)", tok->lexeme); break; }
    default:                   { emit("unknown"); break; }
  }
}
```
the `emit` function can decide what to do with the fragments--make copies, print to console, ignore them, `rm -rf /`, you name it. the ownership is clear here

however, this actually makes the call site quite ugly:
```c
emit(printf, ident1);
printf(" ");
emit(printf, ident2);
printf("\n");
```
breaking the format string `"%s %s\n"` up makes it much harder to make out whats being printed here. it is unacceptable to have cake but not be able to eat it. let us dream up a perfect api:
```c
print(f"{ident1} {ident2}\n");
```

alas, c is not a based language like python. let us cut back just a tad bit:
```c
print("{token} {token}\n", ident1, ident2);
```

we can spam our "strategy pattern" move here again, except this time at runtime instead of compile time. define a formatter to be:
```c
typedef int (*emit_func)(const char *fmt, ...);
typedef void (*fmt_func)(emit_func emit, va_list ap);
typedef struct formatter formatter;
struct formatter {
  const char *spec;
  fmt_func fmt;
  formatter *next;
};
```

we can then define a global formatter registry:
```c
formatter FORMATTERS = {0};

void add_formatter(const char *spec, fmt_func fmt) {
  formatter *f = calloc(1, sizeof(formatter));
  f->spec = spec;
  f->fmt = fmt;
  f->next = FORMATTERS.next;
  FORMATTERS.next = f;
}
```

then we can implement a string interpolation engine that takes an emitter and uses the global formatter registry to interpolate arbitrary objects into strings:
```c
void vemitf(emit_func emit, const char *fmt, va_list ap) {
  while (*fmt) {
    if (*fmt == '{') {
      char spec_buf[BUF_LEN];
      const char *spec_start = ++fmt;
      while (*fmt && *fmt != '}') fmt++;
      assert(*fmt);  // must find closing brace
      const size_t spec_len = fmt - spec_start;
      memcpy(spec_buf, spec_start, spec_len);
      spec_buf[spec_len] = '\0';
      fmt++;

      formatter *f = FORMATTERS.next;
      while (f) {
        if (!strcmp(spec_buf, f->spec)) {
          f->fmt(emit, ap);
          break;
        }
        f = f->next;
      }
      assert(f, "unknown spec: %s", spec_buf);
    } else emit("%c", *fmt++);
  }
}

void print(const char *fmt, ...) {
  va_list ap;
  va_start(ap, fmt);
  vemitf(printf, fmt, ap);
  va_end(ap);
}
```

another thing to note is that our `emit_func`, which, in sane notation, is `(const char *fmt, ...) -> int`, doesnt exactly work for use cases such as dumping to a dynamic string buffer created on the stack at the call site. let us use a "fat pointer" instead, so we can optionally carry a context around:
```c
typedef struct {
  int (*emit)(void *ctx, const char *fmt, ...);
  void *ctx;
} emitter;

void emit(emitter e, const char *fmt, ...) {
  e.emit(e.ctx, fmt, ...);  // pretend this syntax works... sigh, c
}

void emit_print(void *ctx, const char *fmt, ...) {
  printf(fmt, ...);
}

void emit_sb_append(void *ctx, const char *fmt, ...) {
  str_builder *sb = (str_builder *) ctx;
  sb_appendf(sb, fmt, ...);
}

emit((emitter) { .emit = emit_print },
     "hello %s\n", "world");

str_builder sb = sb_create(256);
emit((emitter) { .emit = emit_sb_append, .ctx = &sb },
     "hello %s\n", "world");
```

and that is the story of quasi-f-strings in c. see: [cfmt](https://github.com/hljeong/cfmt)

p.s. im such a negative nancy when i drop the "math131bh homework" register
