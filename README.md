# shcd_2017
为什么上传不了文件了那？？？
# -*- coding: utf-8 -*-
import sys
reload(sys)
sys.setdefaultencoding('utf-8')

import jieba_fast as jieba
import jieba.posseg as pseg
import jieba.analyse
import logging
import codecs
import Message
import json
from gensim import corpora, models, similarities
from pymongo import MongoClient

stop_words = 'CNstopwords.txt'
stopwords = codecs.open(stop_words,'r',encoding='utf8').readlines()
stopwords = [ w.strip() for w in stopwords]

def tokenization(text):
	result = []
	# input texts
	words = jieba.cut(text,cut_all=False)
	for w in words:
		if w not in stopwords:
			result.append(w)
	return result

if __name__ =="__main__":
	out_type1 = 1
	out_type2 = "hdtz"
	messages=Message.Message.getMessages(out_type1,out_type2)
	print(messages.__len__())

    #content divide && get keywords
	out_title = []
	out_content = []
	out_keywords = []
	for m in messages:
		out_title.append(m.title)
		out_content.append(tokenization(m.content))
		out_keywords.append(jieba.analyse.extract_tags(m.content,topK=5))

	#create dictionary
	dictionary = corpora.Dictionary(out_content)

	#doc2bow
	doc_vectors = [dictionary.doc2bow(text) for text in out_content]

	#tf-idf
	tfidf = models.TfidfModel(doc_vectors)
	tfidf_vectors = tfidf[doc_vectors]

	#LSI
	lsi = models.LsiModel(tfidf_vectors, id2word=dictionary, num_topics=2)
	out_vector = lsi[tfidf_vectors]

	#output to mongodb
	connection = MongoClient('127.0.0.1',27017)
	db = connection.testdb
	sets = db.test_dalian
	for i in range(100):
		cont = json.dumps(out_content[i])
		keys = json.dumps(out_keywords[i])
		vec = json.dumps(out_vector[i])
		
		sets.insert({"id":i
			,"exchange":out_type1
			,"type":out_type2
			,"title":out_title[i]
			,"content":cont
			,"keywords":keys
			,"vector":vec})

	connection.close()

'''
output types:
1.out_type1--int--which exchange
2.out_type2--str--news or announcement
3.out_title--string
4.out_content--list of string,divided words
5.out_keywords--list of string,5 keywords
6.out_vector--list of double
'''

