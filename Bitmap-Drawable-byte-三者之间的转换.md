---
title: 'Bitmap,Drawable,byte[] 三者之间的转换'
date: 2016-04-14 14:57:25
tags:
---

### Bitmap->Drawable ###

	private Drawable transBitmapToBitmapDrawable(Bitmap bitmap){
    	BitmapDrawable bd=new BitmapDrawable(getResources(),bitmap); 
        return bd;
    }


> BitmapDrawable是Drawable的子类。

### Drawable->Bitmap ###

	private void drawableToBitamp(Drawable drawable){
         BitmapDrawable bd = (BitmapDrawable) drawable;
         bitmap = bd.getBitmap();
    }


### Bitmap->byte[ ] ###

	private byte[] transBitmapToByteArr(Bitmap bitmap){
	    	ByteArrayOutputStream baos=new ByteArrayOutputStream();
	    	bitmap.compress(Bitmap.CompressFormat.PNG, 100, baos);
	    	return baos.toByteArray();
	}

在写这边文章的时候，有个小疑惑，为什么用的是 **ByteArrayOutputStream**，而不是 **ByteArrayInputStream**，因为 这个输入和输出是相对的概念。过程的执行体是 **bitmap**，当将 **bitmap** 的数据输出到 **byte[]** 这个过程，相对于 **bitmap** 来说是个输出的过程，所以用 **ByteArrayOutputStream** ， 这是我的理解。


### byte[ ] -> Bitmap ###
	
	private Bitmap transByteArrToBitmap(byte[] bytes){
    	return BitmapFactory.decodeByteArray(bytes, 0, bytes.length);
    }