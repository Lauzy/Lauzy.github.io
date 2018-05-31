---
title: 自定义View：Android 歌词控件
date: 2018-05-31 14:42:27
tags: 
	- Android自定义View
---


## 简介

之前做 [TicktockMusic](https://github.com/Lauzy/TicktockMusic) 音乐播放器，一个必要的需求肯定是歌词，在 github 上找了几个，发现或多或少都有点不满足需求，所以就自己动手写了一个。

先附上项目地址和效果图：

地址：[https://github.com/Lauzy/LyricView](https://github.com/Lauzy/LyricView)

效果图：

<img src="http://oop6dcmck.bkt.clouddn.com/20180531LrcViewCompress.gif" width = "270" height = "450" alt="效果图"/>


#### 需求
歌词的需求我想大家都很清楚，简单的话，直接打开一个音乐播放器查看一下。我们打开后分析一下歌词的功能：歌词完整的显示出来、当前歌词变色、可以根据时间而进行定位、可以手动滑动、滑动后显示一个指示器、点击指示器播放进度跳转、滑动时指示器变色等等。OK，我们自己写歌词控件，这些功能也是必不可少的，接下来就逐步分析下实现的过程。


## 实现

- 歌词解析
- 歌词显示
- 滑动处理


基本实现就是这三个过程，接下来一步步的分析。

#### 歌词解析

首先，我们在网上下载一个歌词，即以 lrc 为后缀的文件。比如海阔天空这首歌的歌词，我们用记事本或者其他工具打开后就可以看到具体的歌词内容，如下：

```

[ti: 海阔天空]
[ar:黄家驹]
[al:乐与怒]
[by:mp3.50004.com]
[00:00.00]Beyond：海阔天空 
[01:40.00][00:16.00]今天我寒夜里看雪飘过 
[01:48.00][00:24.00]怀著冷却了的心窝飘远方 
[01:53.00][00:29.00]风雨里追赶 
...

[00:42.00]多少次迎著冷眼与嘲笑 
[00:49.00]从没有放弃过心中的理想 
[00:54.00]一刹那恍惚 
...

```

可以看到，歌词主要包含歌名、歌手、专辑、作者等头元素，以及歌词的主体内容，我们需要处理的就是主体的歌词内容。首先，歌词是一行一行的文本，其次，每行的文本都包含时间标签和具体的一行歌词，我们首先将歌词解析为一行行的数据。

```java

        InputStreamReader isr = null;
        BufferedReader br = null;
        try {
            isr = new InputStreamReader(inputStream, CHARSET);
            br = new BufferedReader(isr);
            String line;
            while ((line = br.readLine()) != null) {
                //此处的 line 即为一行行的文本
                //parseLrc 方法为解析单行
                List<Lrc> lrcList = parseLrc(line);
                if (lrcList != null && lrcList.size() != 0) {
                    lrcs.addAll(lrcList);
                }
            }
            sortLrcs(lrcs);
            return lrcs;
        }catch ...

```

解析为一行行的文字后，就需要具体的处理单行的文字了，我们可以看到，大部分歌词包含两种格式，即单个时间标签和多个时间标签，这里可以采用正则表达式来匹配文字，正则表达式为 ((\[\d{2}:\d{2}\.\d{2}])+)(.*) 

```

[01:53.00][00:29.00]风雨里追赶    //多个时间标签

[00:42.00]多少次迎著冷眼与嘲笑     //单个时间标签

```

接下来根据正则表达式来解析单行歌词

```java

    private static List<Lrc> parseLrc(String lrcLine) {
        if (lrcLine.trim().isEmpty()) {
            return null;
        }
        List<Lrc> lrcs = new ArrayList<>();
        Matcher matcher = Pattern.compile(LINE_REGEX).matcher(lrcLine);
        if (!matcher.matches()) {
            return null;
        }

        String time = matcher.group(1);
        String content = matcher.group(3);
        Matcher timeMatcher = Pattern.compile(TIME_REGEX).matcher(time);

        while (timeMatcher.find()) {
            String min = timeMatcher.group(1);
            String sec = timeMatcher.group(2);
            String mil = timeMatcher.group(3);
            Lrc lrc = new Lrc();
            if (content != null && content.length() != 0) {
                lrc.setTime(Long.parseLong(min) * 60 * 1000 + Long.parseLong(sec) * 1000
                        + Long.parseLong(mil) * 10);
                lrc.setText(content);
                lrcs.add(lrc);
            }
        }
        return lrcs;
    }

```
这样，第一步就完成了，歌词解析完成后得到歌词的数据集合，每个元素都包括时间和内容。

#### 歌词显示

歌词显示的思路就是将歌词一行行的画出来，我们首先假设屏幕足够大，那么只需要定位第一行歌词的位置，画出来第一行歌词，然后逐行下移一个固定的距离，再画出下一行歌词，依次类推，整个歌词内容就会全部花在画布上了。依照这个思路，我们可以先画出来文字。

```java

      //此处为伪代码

      float y = getLrcHeight() / 2;
      float x = getLrcWidth() / 2 + getPaddingLeft();
      for (int i = 0; i < getLrcCount(); i++) {
            if (i > 0) {
                y += textHeight  + mLrcLineSpaceHeight;
            }
          ...
           canvas.drawText(text, x, y, mPaint);
       }

```

画出来文字的思路就是这样，首先从屏幕的中间开始，然后纵坐标每次增加文字的高度与距离之和，依次画出来每行文字。