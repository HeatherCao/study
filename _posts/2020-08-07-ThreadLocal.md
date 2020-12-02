---
layout: post
title: java——ThreadLocal
keywords: java, Thread
description: ThreadLocal的使用
categories: [java]
tags: [java]
---

*  目录
{:toc}

# 定义
	@Component
	public class MailTheadLocal extends ThreadLocal<Object> {

		public Object getVal(){
			return super.get();
		}

		public void put(Object val) {
			super.set(val);
		}
	}

# 使用
	public static MailTheadLocal mailTheadLocal = new MailTheadLocal();
	
	## set
	mailTheadLocal.put(Arrays.asList(jobName, "manually started", operator));
	
	## get
	List<String> listParam = (List) ThreadLocalUtil.mailTheadLocal.get();

	
	

	