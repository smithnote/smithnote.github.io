---
layout: archive
title: 情侣随机匹配
data: 2017-12-14 11:00
categories: programming
tag: match

---
前段时间，朋友的学弟他们脑动大开，想要在校园里举办一次7天情侣匹配活动，这个点子让报名参加的人数不少，每位报名参加的学生都填选了性格，和学年以及其他个人信息，活动举办方要根据这些信息让男女随机配对，但是随机配对的前提是尽量性格和学年相同，这样能够保证成为情侣的概率大增，至少不会是尬聊。如何能达到这样的匹配效果呢？  
观察参选人的信息，只有性别，学年，性格这三项信息有用，所以这里有个想法，如果我们可以建立一个三维数组，第一维表示性别，第二维表示学年，第三维表示性格(这里只有三种类型，要是种类多了，就得用向量表示了，我们这里简化了)，之后，我们可以将性别，学年，性格都相同的放置到对应的位置（对应的位置里面是个set,能够容纳多个数据）, 接下来就简单了，直接从性别不同，学年性格相同的维度里面随机取数据配对就行。代码实现如下
```
#!/usr/bin/python
#coding:utf-8

# Copyright (c) 2017 smtihemail@163.com. All rights reserved.
# Author:smithemail@163.com
# Time  :2017-04-25

#
#   思路：因为之考虑两个要素：兴趣爱好和年龄，两者的类别分别有4种，
#         故可以映射到一个4×4数组中，数组的横坐标代表兴趣爱好，纵
#         左边代表学年段，数组的元素是一个list, list中存放着相同
#         兴趣爱好和学年段的学生，男女各映射到来那个4×4数组中，
#         而后随机选取两个4×4数组（男和女）中相同下标list中的学生。
#         一轮过后，放开条件，或兴趣爱好相同但学年不同，或学年相同
#         但兴趣爱好不同



import sys
import pdb
import random
import pymysql.cursors
from optparse import OptionParser

parser = OptionParser()
parser.add_option("-H", "--host", type="string", dest="host", default="localhost", help="mysql host")
parser.add_option("-P", "--port", type="int", dest="port", default=3306, help="mysql port")
parser.add_option("-u", "--user", type="string", dest="user", default="root", help="mysql user")
parser.add_option("-p", "--passwd", type="string", dest="passwd", default="root", help="mysql passwd")
parser.add_option("-d", "--db", type="string", dest="db", default="datadb", help="mysql datadb")
parser.add_option("-o", "--output", type="string", dest="out", default="result.txt", help="result")
(options, args) = parser.parse_args()

class Student :
    def __init__(self, sid, name, sex, hobby):
        self.sid_ = sid
        self.name_ = name
        self.sex_ = sex
        self.hobby_ = hobby

class MatchFriend :
    def __init__(self, host, port, user, passwd, db):
        self.host_ = host
        self.port_ = port
        self.user_ = user
        self.passwd_ = passwd
        self.connectdb_ = None
        self.db_ = db;
        self.male_members_ = list()
        self.female_members_ = list()
        self.fp = open(options.out, 'a')
        self.fp.write("formate: sid, name, sex, age\n")
    def __del__(self) :
        self.connectdb_.close()
        self.fp.close()
    def connectdb(self):
        try:
            self.connectdb_ = pymysql.connect(host=self.host_, port=self.port_,
                                              user=self.user_, password=self.passwd_,
                                              db=self.db_)
        except Exception, e:
            print "connect db failed: ", e.message
            return -1
        return 0
    def match(self):
        if self.connectdb() != 0:
            print "connectdb fail, and exit";
            return -1;
        for i in xrange(4) :
            self.male_members_.append(list())
            self.female_members_.append(list())
            for j in xrange(4) :
                self.male_members_[i].append(list())
                self.female_members_[i].append(list())
        with self.connectdb_.cursor() as cursor:
            sql_get_info = "select id, name, sex, hobby from cp_data"
            cursor.execute(sql_get_info)
            student_data = cursor.fetchall()
            for sinfo in student_data :
                # 将数据放入4*4二维数组中
                student = Student(sinfo[0], sinfo[1], sinfo[2], sinfo[3])
                i = sinfo[3] -1
                j = 2016 - int(sinfo[0][:4])
                if j > 3 : j = 3 # 还有2012届的
                if j < 0 : j = 0 # 还有乱填的
                if student.sex_ == 0:
                    self.female_members_[i][j].append(student)
                else :
                    self.male_members_[i][j].append(student)
        self.rand_match()
        return 0
    def rand_match(self) :
        # @brief:随机匹配
        # first match: 基准是兴趣爱好相同
        for gap in xrange(3) : # 3代表年级段最多相差3-1
            for i in xrange(4) :
                for j in xrange(4) :
                    k = j
                    while len(self.male_members_[i][k]) == 0 and k+1 < 4 and k < j+gap : k += 1
                    male_list = self.male_members_[i][k];
                    k = j
                    while len(self.female_members_[i][k]) == 0 and k+1 < 4 and k < j+gap : k += 1
                    female_list = self.female_members_[i][k];
                    while len(male_list) and len(female_list):
                        male = random.choice(male_list)
                        female = random.choice(female_list)
                        self.output(male, female)
                        male_list.remove(male)
                        female_list.remove(female)
        # second match: 基准是年龄段相同
        for gap in xrange(2) :
            for i in xrange(4) :
                for j in xrange(4) :
                    k = i
                    while len(self.male_members_[k][j]) == 0 and k+1 < 4 and k < i+gap : k += 1
                    male_list = self.male_members_[k][j];
                    k = i
                    while len(self.female_members_[k][j]) == 0 and k+1 < 4 and k < i+gap : k += 1
                    female_list = self.female_members_[k][j];
                    while len(male_list) and len(female_list):
                        male = random.choice(male_list)
                        female = random.choice(female_list)
                        self.output(male, female)
                        male_list.remove(male)
                        female_list.remove(female)
    def output(self, male, female) :
        self.fp.write("%s, %s, %d, %d\n" % (male.sid_, male.name_, male.sex_, male.hobby_))
        self.fp.write("%s, %s, %d, %d\n" % (female.sid_, female.name_, female.sex_, female.hobby_))
        self.fp.write("\n")
        self.fp.flush()

if __name__ == "__main__" :
    match_friend = MatchFriend(options.host, options.port, options.user, options.passwd, options.db);
    match_friend.match()
```
以上只是对维度比较少的数据通用，如果维度多了，也要求随机匹配，这不就是聚类了吗？这也就涉及到机器学习相关的知识了，本人暂时了解不深，就点到为止吧，后续进修了，再聊吧！
