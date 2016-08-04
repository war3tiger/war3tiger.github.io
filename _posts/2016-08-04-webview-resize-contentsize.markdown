---
layout: post
title:  "UIWebView contentSize"
date:   2016-08-04 16:56:31 +0800
categories: jekyll update
---

### UIWebView contentSize change

UIWebView在加载内容时有时候需要改变内容的字体大小，这个时候UIWebView的scrollView中的contentSize没有变化，解决该问题的方法如下：

	-(void)webViewDidFinishLoad:(UIWebView *)webView
	{
   		[SVProgressHUD dismiss];

		UserManagerFontType fontType = [UserManager getSystemFontType];
		NSArray *array = @[@"80%", @"100%", @"120%"];
		NSString *str = [NSStringstringWithFormat:@"document.getElementsByTagName('body')[0].style.webkitTextSizeAdjust= '%@'", array[fontType -  UserManagerFontTypeSmall]];
    	[webView stringByEvaluatingJavaScriptFromString:str];
    
		CGRect webViewFrame = webView.frame;
    	webViewFrame.size.height = 1;
    	webView.frame = webViewFrame;
    	CGSize fittingSize = [webView sizeThatFits:CGSizeZero];
    	webViewFrame.size = fittingSize;
    	webView.frame = webViewFrame;
	}
