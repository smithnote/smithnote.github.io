---
layout: archive
title: UTF-8汉字转拼音和简拼
date: 2018-01-04 11:15
categories: programming
tag: utf8 chinese pinyin

---
在工作中遇到了汉字转换为拼音的需求，看了相关原有的代码，又到网上去找了
相关的资料文档博客等，总结一下汉字与拼音之间的转换，本人也写了几个demo
来验证

## 汉字转换为拼音

汉字转换为拼音无非就是一步查询工作，如何查询快速是衡量程序的优秀性
1. 查询数据库，对汉字字段建立索引，这样的话复杂复复杂度logN, 
[汉字总共大概有8万多个](http://www.hwjyw.com/resource/content/2012/02/09/23489.shtml)
这样查询的数度也挺快。优点是客户端简化。缺点是依赖性强，要求每个客户端都能连接数据库。

2. 查询程序内的变量
    * 使用map读取汉字与拼音之间的映射，查询的时候直接查询map,这样的话速度取决于map的性能和内部实现方式，
    不过考虑到汉字的数量并不是特别多，直接查询的话复杂度LogN（map内部实现红黑树）和O(1)（map内部实现hash）差别不大,
    所以查询速度不是问题。所以优点是客户端独立性强，缺点是需要占用部分内存
    * 针对上面的方法做优化，如何减小内存占用量，map内部的实现（各种指针额外参数）显然没有太多的简化空间，所以只能针对key, value来进行。
    key也就是汉字，以string存的话，通常占用两个字节(utf-8编码或GBK编码),也没是呢没好办法来优化。
    value是拼音字符串, 针对都是由26个字母组成，所以5bits就可以表示一个字符并且最长的拼音长度是6个字符，所以最多4个uint8_t便可以
    表示一个拼音，优化的理论值是(1-5/8)(原先8bit一个字符)，不过考虑到代码实现的问题，只能接近这个值。
    这样下来能够节省一小部分内存，显然效果不是特别明显，而且增加了编码与解码时间还有代码的可读.
    
    * 既然key-value的值优化的空不大只能考虑map映射来优化了，单纯的换map库是没多大用的。我们注意到中文的汉字存在一个unicode国际码，
    而且是连续的，这样可以换个思路直接用一个大的数组来存储所有的拼音，汉字转换为unicode后在转换为数组的下标（也就是减去最小的汉字的unicode值）
    这样的'map'的额外的消耗空间几乎没有，内存的利用率大大增加, 实现如下（篇幅有限就不写全那个大的拼音数组了）:

    ```
    int utf8_to_unicode(const std::string &word) {
        if (word.empty()) {
            return 0;
        }
        if (((unsigned char)word[0] & 0x80 )== 0) {
            return (int)word[0];
        } else if ((unsigned char)(word[0] & 0xE0) == 0xC0) {
            if (word.size() < 2) {
                return 0;
            }
            return (((int)word[0] & 0x1F) << 6) | ((int)word[1] & 0x3F);
        } else if ((unsigned char)(word[0] & 0xF0) == 0xE0) {
            if (word.size() < 3) {
                return 0;
            }
            return (((int)word[0] & 0x0F) << 12)
                  |(((int)word[1] & 0x3F) << 6)
                  | ((int)word[2] & 0x3F);
        } else if ((unsigned char)(word[0] & 0xF8) == 0xF0) {
            if (word.size() < 4) {
                return 0;
            }
            return (((int)word[0] & 0x07) << 18)
                  |(((int)word[1] & 0x3F) << 12)
                  |(((int)word[2] & 0x3F) << 6)
                  | ((int)word[3] & 0x3F);
        } else if ((unsigned char)(word[0] & 0xFC) == 0xF8) {
            if (word.size() < 5) {
                return 0;
            }
            return (((int)word[0] & 0x03) << 24)
                  |(((int)word[1] & 0x3F) << 18)
                  |(((int)word[1] & 0x3F) << 12)
                  |(((int)word[2] & 0x3F) << 6)
                  | ((int)word[3] & 0x3F);
        } else if ((unsigned char)(word[0] & 0xFE) == 0xFC) {
            if (word.size() < 6) {
                return 0;
            }
            return (((int)word[0] & 0x01) << 30)
                  |(((int)word[1] & 0x3F) << 24)
                  |(((int)word[1] & 0x3F) << 18)
                  |(((int)word[1] & 0x3F) << 12)
                  |(((int)word[2] & 0x3F) << 6)
                  | ((int)word[3] & 0x3F);
        } else {
            return 0;
        }
    }
    bool utf8_to_pinyin(const string &word, string &pinyin) {
        if (word.empty()) {
            return false;
        }
        int unicode = utf8_to_unicode(word);
        // 判断是否在汉字的范围，由pinyin_list的长度决定
        if (unicode > 19968 && unicode < 40870) { 
            pinyin = pinyin_list[unicode-19968];//拼音数组
            return true;
        } else {
            return false;
        }
    }
    ```
    * 关于多音字的支持。解决了汉字转拼音这一步后，有时候要多字的支持，这个时候我们可以在上面返回拼音的地方
    稍微修改一下，返回该汉字所有的拼音（使用特殊字符间隔），然后我们要做的就是对给点的一条汉字字符串返回所有
    拼音排列组合，相应的代码如下：

    ```
    bool hz2py(std::string words, std::vector<std::string> &pinyin) {
        if (words.empty()) {
            return false;
        }
        std::vector<std::vector<string> >pinyin_vec;
        //每个单词一个vector, vector存每个单词的所有拼音
        get_pinyin_vec(words, pinyin_vec);
        pinyin.clear();
        pinyin.push_back("");
        for (size_t i=0;i<pinyin_vec.size();i++){
            std::vector<string> &vec = pinyin_vec[i];
            size_t py_size = pinyin.size();
            for (size_t j = 0; j < py_size; ++j) {
                std::string pre_pinyin = pinyin[j];
                if (pinyin.size() > 32) {
                    vec.resize(1);// 实际使用中做下限制
                }
                for (size_t k = 0; k < vec.size(); ++k) {
                    if (k == 0) {
                        pinyin[j].append(vec[k]);
                    } else {
                        pinyin.push_back(pre_pinyin + vec[k]);
                    }
                }
            }
        }
        return true;
    }

    ```
    **Note**:实际使用过程中有必要限制下大小的，假设提供的中文字符串是几十个多音字,每个多音字
    有两个选项，2的几十次方的空间需求，程序会直接挂掉，所以最好加个限制。

## 汉字转简拼

跟上面的汉字转拼音一样，不过这次数据了量更小了,value值是单纯的一个字符，很容易
想到直接利用上面的方法转换成拼音后再直接取首字母得到汉字的简拼了。

##### PS：不过听闻GBK编码的汉字十分有规律，能够更简单的进行汉字转拼音和简拼，有时间再研究看看
