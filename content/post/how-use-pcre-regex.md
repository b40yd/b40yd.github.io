+++
title = "How Use Pcre Regex"
date = 2025-04-09
lastmod = 2025-04-11T15:35:24+08:00
tags = ["c/c++", "pcre"]
categories = ["c/c++", "pcre"]
draft = false
author = "B40yd"
+++

## 实现正则表达式 {#实现正则表达式}

如何使用c语言，使用pcre实现正则表达式解析，要求支持\`//igms\`结构。

参考PHP实现的正则表达式匹配 ，支持\`//imgs\`多模式匹配语法，比如同时支持单行 多行，忽略大小写，全局等。

例如：

```nil
/^(?:webhook-[a-zA-Z0-9]+|x-[a-zA-Z0-9]+/igms
```

匹配测试如下：

```nil
x-count
x- xxx
xx-xxx
webhook-id
webhook-event
w ebhook event
web hook-event
```


## 解析正则表达式 {#解析正则表达式}

```c
#include <stdio.h>
#include <string.h>
#include <pcre.h>
#include <stdbool.h>
#include <ctype.h>

#define MAX_PATTERN_LENGTH 1000
#define MAX_SUBJECT_LENGTH 1000
#define MAX_MATCHES 100

typedef struct {
    int start;
    int end;
} match_t;

int match_regex(const char* regex, const char* subject, match* matches, int max_matches) {
    char pattern[MAX_PATTERN_LENGTH];
    int options = 0;
    const char* error;
    int erroffset;
    pcre* re;
    int rc;
    int ovector[MAX_MATCHES * 3];
    int match_count = 0;
    bool global = false;

    // 解析正则表达式和选项
    const char* pattern_start = strchr(regex, '/');
    const char* pattern_end = strrchr(regex, '/');

    if (pattern_start == NULL || pattern_end == NULL || pattern_start == pattern_end) {
        printf("Invalid regex format. Use /pattern/options\n");
        return -1;
    }

    // 提取模式
    int pattern_length = pattern_end - pattern_start - 1;
    strncpy(pattern, pattern_start + 1, pattern_length);
    pattern[pattern_length] = '\0';

    // 解析选项
    const char* options_start = pattern_end + 1;
    while (*options_start) {
        switch (tolower(*options_start)) {
            case 'i': options |= PCRE_CASELESS; break;
            case 'm': options |= PCRE_MULTILINE; break;
            case 's': options |= PCRE_DOTALL; break;
            case 'x': options |= PCRE_EXTENDED; break;
            case 'g': global = true; break;
            default:
                printf("Unknown option: %c\n", *options_start);
                return -1;
        }
        options_start++;
    }

    // 编译正则表达式
    re = pcre_compile(pattern, options, &error, &erroffset, NULL);
    if (re == NULL) {
        printf("PCRE compilation failed at offset %d: %s\n", erroffset, error);
        return -1;
    }

    // 执行匹配
    int subject_length = strlen(subject);
    int offset = 0;

    do {
        rc = pcre_exec(re, NULL, subject, subject_length, offset, 0, ovector, MAX_MATCHES * 3);

        if (rc < 0) {
            if (rc != PCRE_ERROR_NOMATCH) {
                printf("Matching error %d\n", rc);
            }
            break;
        }

        for (int i = 0; i < rc && match_count < max_matches; i++) {
            matches[match_count].start = ovector[i*2];
            matches[match_count].end = ovector[i*2+1];
            match_count++;
        }

        offset = ovector[1];

    } while (global && offset < subject_length && match_count < max_matches);

    // 释放编译后的正则表达式
    pcre_free(re);

    return match_count;
}


void print_matches(const char* subject, match_t* matches, int match_count) {
    for (int i = 0; i < match_count; i++) {
        printf("match %d: %.*s\n", i + 1, matches[i].end - matches[i].start, subject + matches[i].start);
    }
}

int main() {
    char regex[MAX_PATTERN_LENGTH];
    char subject[MAX_SUBJECT_LENGTH];
    match_t matches[MAX_MATCHES];
    int match_count;

    printf("Enter regex (in /pattern/options format): ");
    fgets(regex, sizeof(regex), stdin);
    regex[strcspn(regex, "\n")] = 0;  // 移除换行符

    printf("Enter subject string: ");
    fgets(subject, sizeof(subject), stdin);
    subject[strcspn(subject, "\n")] = 0;  // 移除换行符

    match_count = match_regex(regex, subject, matches, MAX_MATCHES);

    if (match_count >= 0) {
        printf("Found %d matches:\n", match_count);
        print_matches(subject, matches, match_count);
    }

    return 0;
}
```
