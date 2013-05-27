---
layout: post
title: "Catching Logs"
date: 2013-05-26 10:18
comments: true
categories: 
published: false
---

{% codeblock LogCatcher.m lang:objc %}
#import <Foundation/Foundation.h>

@interface LogCatcher : NSObject
+(id)getLogs;
+(void)resetLogs;
+(BOOL)containsLogWithFormatString:(NSString *)fmt;
+(BOOL)containsLogWithFullString:(NSString *)full;
+(BOOL)containsLogWithFullStringMatching:(NSString *)full;
+(BOOL)containsLogWithFormatStringMatching:(NSString *)full;
+(BOOL)containsLogWithFormatString:(NSString *)fmt fullString:(NSString *)full;
+(BOOL)containsLogWithFormatStringMatching:(NSString *)fmt fullStringMatching:(NSString *)full;
+(NSNumber *)numberOfPlaceholdersInFormatString:(NSString *)fmt;
@end

void addLog(NSString *format, ...) NS_FORMAT_FUNCTION(1,2);
{% endcodeblock %}
