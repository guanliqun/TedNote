---
layout: post
title: Redis3.03代码阅读(1)-adlist代码分析
category: 技术
comments: true
---

#1，概述
本文作为redis代码阅读的笔记。以下内容主要是针对adlist.h/adlist.c文件中的list链表进行代码分析和注释。
总体上来说该adlist有如下几个特点：
1，通过c语言实现了泛型的list链表，同时实现了iterator迭代器。虽然简单，但是很巧妙。
2，整个实现的内存都是通过堆分配，
3，通过宏封装了部分的功能代码，增加了代码的可阅读性；
但是编码风格还是有不是很完备的地方。比如所有指针参数入口都不判断指针是否为空，默认都是有效的指针参数，这并不是很好的变成风格。
当然如果仅仅是redis内部使用的数据结构，并且作者很有把握，这样写代码能省略部分错误检查的开销。

闲话少说，下面分别对adlist.h和adlist.c 代码进行中文分析注释说明。

#2，adlist.h
```C++
  1 /* adlist.h - A generic doubly linked list implementation
  2  *
  3  * Copyright (c) 2006-2012, Salvatore Sanfilippo <antirez at gmail dot com>
  4  * All rights reserved.
  5  *
  6  * Redistribution and use in source and binary forms, with or without
  7  * modification, are permitted provided that the following conditions are met:
  8  *
  9  *   * Redistributions of source code must retain the above copyright notice,
 10  *     this list of conditions and the following disclaimer.
 11  *   * Redistributions in binary form must reproduce the above copyright
 12  *     notice, this list of conditions and the following disclaimer in the
 13  *     documentation and/or other materials provided with the distribution.
 14  *   * Neither the name of Redis nor the names of its contributors may be used
 15  *     to endorse or promote products derived from this software without
 16  *     specific prior written permission.
 17  *
 18  * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
 19  * AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 20  * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 21  * ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
 22  * LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
 23  * CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
 24  * SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
 25  * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
 26  * CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
 27  * ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
 28  * POSSIBILITY OF SUCH DAMAGE.
 29  */
 30 
 31 #ifndef __ADLIST_H__
 32 #define __ADLIST_H__
 33 
 34 /* Node, List, and Iterator are the only data structures used currently. */
 35 
 36 /* 节点信息 */
 37 // 每个listNode都有一个向前、向后的指针，以及一个保存任何数据类型void指针。这样设计的好处就是链表可以保持任何数据类型
 38 // 很灵活，堪比C++的list模板，listNode甚至在list上可以分别保存不同的数据类型
 39 typedef struct listNode {
 40     struct listNode *prev;
 41     struct listNode *next;
 42     void *value;
 43 } listNode;
 44 
 45 /* 迭代器 */
 46 // C语言实现的迭代器，一个listNode的指针，一个迭代方向
 47 typedef struct listIter {
 48     listNode *next;
 49     int direction;
 50 } listIter;
 51 
 52 /* list链表 */
 53 typedef struct list {
 54     listNode *head;                 //链表头指针
 55     listNode *tail;                 //链表尾指针
 56     void *(*dup)(void *ptr);        //dup函数指针
 57     void (*free)(void *ptr);        //free函数指针
 58     int (*match)(void *ptr, void *key); //match函数指针
 59     unsigned long len;              //链表长度
 60 } list;
 61 
 62 /* Functions implemented as macros */
 63 // 用宏实现的一些基本功能 */
 64 // 宏：编译时期就展开代码，没任何运营是开销，为啥要这样做？好处就是增加代码的可阅读性
 65 
 66 #define listLength(l) ((l)->len)            //获取list长度
 67 #define listFirst(l) ((l)->head)            //获取list头部
 68 #define listLast(l) ((l)->tail)             //获取list尾部
 69 #define listPrevNode(n) ((n)->prev)         //获取某个节点的前置
 70 #define listNextNode(n) ((n)->next)         //获取某个节点的后置
 71 #define listNodeValue(n) ((n)->value)       //获取某给节点的值,这里的value不是值，而是函数指针？
 72 
 73 #define listSetDupMethod(l,m) ((l)->dup = (m))          //设置列表的dup方法
 74 #define listSetFreeMethod(l,m) ((l)->free = (m))        //设置列表的free方法
 75 #define listSetMatchMethod(l,m) ((l)->match = (m))      //设置列表的match方法
 76 
 77 #define listGetDupMethod(l) ((l)->dup)                  //获取列表的dup方法
 78 #define listGetFree(l) ((l)->free)                      //获取列表的free方法
 79 #define listGetMatchMethod(l) ((l)->match)              //获取列表的match方法
 80 
 81 /* Prototypes */
 82 list *listCreate(void);
 83 void listRelease(list *list);
 84 list *listAddNodeHead(list *list, void *value);
 85 list *listAddNodeTail(list *list, void *value);
 86 list *listInsertNode(list *list, listNode *old_node, void *value, int after);
 87 void listDelNode(list *list, listNode *node);
 88 listIter *listGetIterator(list *list, int direction);
 89 listNode *listNext(listIter *iter);
 90 void listReleaseIterator(listIter *iter);
 91 list *listDup(list *orig);
 92 listNode *listSearchKey(list *list, void *key);
 93 listNode *listIndex(list *list, long index);
 94 void listRewind(list *list, listIter *li);
 95 void listRewindTail(list *list, listIter *li);
 96 void listRotate(list *list);
 97 
 98 /* Directions for iterators */
 99 #define AL_START_HEAD 0
100 #define AL_START_TAIL 1
101 
102 #endif /* __ADLIST_H__ */
```
                                                                               
                                                                               
#3，adlist.c
```C++
  1 /* adlist.c - A generic doubly linked list implementation
  2  *
  3  * Copyright (c) 2006-2010, Salvatore Sanfilippo <antirez at gmail dot com>
  4  * All rights reserved.
  5  *
  6  * Redistribution and use in source and binary forms, with or without
  7  * modification, are permitted provided that the following conditions are met:
  8  *
  9  *   * Redistributions of source code must retain the above copyright notice,
 10  *     this list of conditions and the following disclaimer.
 11  *   * Redistributions in binary form must reproduce the above copyright
 12  *     notice, this list of conditions and the following disclaimer in the
 13  *     documentation and/or other materials provided with the distribution.
 14  *   * Neither the name of Redis nor the names of its contributors may be used
 15  *     to endorse or promote products derived from this software without
 16  *     specific prior written permission.
 17  *
 18  * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
 19  * AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 20  * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 21  * ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE
 22  * LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
 23  * CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
 24  * SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
 25  * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
 26  * CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
 27  * ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
 28  * POSSIBILITY OF SUCH DAMAGE.
 29  */
 30 
 31 
 32 #include <stdlib.h>
 33 #include "adlist.h"
 34 #include "zmalloc.h"
 35 
 36 /* Create a new list. The created list can be freed with
 37  * AlFreeList(), but private value of every node need to be freed
 38  * by the user before to call AlFreeList().
 39  *
 40  * On error, NULL is returned. Otherwise the pointer to the new list. */
 41 list *listCreate(void)
 42 {
 43     /*从堆里申请一块内存，然后初始化list ,返回该内存地址 */
 44     struct list *list;
 45 
 46     if ((list = zmalloc(sizeof(*list))) == NULL)
 47         return NULL;
 48     list->head = list->tail = NULL;
 49     list->len = 0;
 50     list->dup = NULL;
 51     list->free = NULL;
 52     list->match = NULL;
 53     return list;
 54 }
 55 
 56 /* Free the whole list.
 57  *
 58  * This function can't fail. */
 59 void listRelease(list *list)
 60 {
 61     unsigned long len;
 62     listNode *current, *next;
 63 
 64     current = list->head;    //current指针指向list头第一个节点,从头开始清理
 65     len = list->len;         //list长度
 66     while(len--) {
 67         next = current->next;   //遍历节点
 68         if (list->free) list->free(current->value);
 69         zfree(current);     //这里已经断开了。只能通过next来找到余下来的listNode串
 70         current = next;     //current指向next当前的节点
 71     }
 72     zfree(list);        //最后把list头清理掉
 73 }
 74 
 75 /* Add a new node to the list, to head, containing the specified 'value'
 76  * pointer as value.
 77  *
 78  * On error, NULL is returned and no operation is performed (i.e. the
 79  * list remains unaltered).
 80  * On success the 'list' pointer you pass to the function is returned. */
 81 list *listAddNodeHead(list *list, void *value)
 82 {
 83     listNode *node;
 84 
 85     if ((node = zmalloc(sizeof(*node))) == NULL)        //先申请存储的node空间
 86         return NULL;
 87     node->value = value;
 88     if (list->len == 0) {                               //如果是第一个node
 89         list->head = list->tail = node;                 //list头尾都指向它
 90         node->prev = node->next = NULL;                 //它自己前后节点都是空，因为只有他自己没其他的node
 91     } else {
 92         node->prev = NULL;                              //在list头插入，所以node的前置是空的
 93         node->next = list->head;                        //node的next指向原来list->head指向的地方
 94         list->head->prev = node;                        //原来list->head指向的node，前置指向node
 95         list->head = node;                              //list->head指向新的node
 96     }
 97     list->len++;
 98     return list;
 99 }
100 
101 /* Add a new node to the list, to tail, containing the specified 'value'
102  * pointer as value.
103  *
104  * On error, NULL is returned and no operation is performed (i.e. the
105  * list remains unaltered).
106  * On success the 'list' pointer you pass to the function is returned. */
107 list *listAddNodeTail(list *list, void *value)
108 {
109     listNode *node;
110 
111     if ((node = zmalloc(sizeof(*node))) == NULL)
112         return NULL;
113     node->value = value;
114     if (list->len == 0) {
115         list->head = list->tail = node;
116         node->prev = node->next = NULL;
117     } else {
118         node->prev = list->tail;
119         node->next = NULL;
120         list->tail->next = node;
121         list->tail = node;
122     }
123     list->len++;
124     return list;
125 }
126 
127 list *listInsertNode(list *list, listNode *old_node, void *value, int after) {
128     listNode *node;
129 
130     if ((node = zmalloc(sizeof(*node))) == NULL)                //申请一个node空间
131         return NULL;
132     node->value = value;                                        //赋值
133     if (after) {                                                //after非0,插在old_node后面
134         node->prev = old_node;
135         node->next = old_node->next;
136         if (list->tail == old_node) {
137             list->tail = node;
138         }
139     } else {                                                    //after为0，插在old_node前面
140         node->next = old_node;
141         node->prev = old_node->prev;
142         if (list->head == old_node) {
143             list->head = node;
144         }
145     }                                   //为什么有以下代码？那是因为以上if(after)的逻辑代码只处理了node的前置和后置，
146                                         //而node对应前置和后置的各自后置和前置还指向老的节点，需要修正。因此有了下面
147                                         //的代码
148     if (node->prev != NULL) {                                   //node前置非空，则前置的后置就是node
149         node->prev->next = node;
150     }
151     if (node->next != NULL) {                                   //node后置非空，则后置的前置就是node
152         node->next->prev = node;
153     }
154     list->len++;
155     return list;
156 }
157 
158 /* Remove the specified node from the specified list.
159  * It's up to the caller to free the private value of the node.
160  *
161  * This function can't fail. */
162 void listDelNode(list *list, listNode *node)
163 {
164     if (node->prev)                         //直接把node从list摘除，然后修正node前置和后置的指针即可
165         node->prev->next = node->next;      //前置非空，修正前置的后置指向原来node的后置
166     else
167         list->head = node->next;            //前置为空，则表示node是list头的指针，则需要修正的是head
168     if (node->next)                         //后置非空，修正后置的前置指向原来node的前置
169         node->next->prev = node->prev;
170     else
171         list->tail = node->prev;            //后置为空，则表示node是list尾的指针，则需要修正的是tail
172     if (list->free) list->free(node->value);
173     zfree(node);                            //最后，清理node空间，list长度减1
174     list->len--;
175 }
176 
177 /* Returns a list iterator 'iter'. After the initialization every
178  * call to listNext() will return the next element of the list.
179  *
180  * This function can't fail. */
181 listIter *listGetIterator(list *list, int direction)
182 {
183     listIter *iter;
184 
185     if ((iter = zmalloc(sizeof(*iter))) == NULL) return NULL;           //申请listIter的存储空间
186     if (direction == AL_START_HEAD)         //根据direction方向，分别指向头或者尾
187         iter->next = list->head;
188     else
189         iter->next = list->tail;
190     iter->direction = direction;
191     return iter;
192 }
193 
194 /* Release the iterator memory */
195 void listReleaseIterator(listIter *iter) {      //简单清理listIter存储空间，为什么不直接zfree?那是因为listIter是在函数
196                                                 //内部申请的
197     zfree(iter);
198 }
199 
200 /* Create an iterator in the list private iterator structure */
201 void listRewind(list *list, listIter *li) {
202     li->next = list->head;
203     li->direction = AL_START_HEAD;
204 }
205 
206 void listRewindTail(list *list, listIter *li) {
207     li->next = list->tail;
208     li->direction = AL_START_TAIL;
209 }
210 
211 /* Return the next element of an iterator.
212  * It's valid to remove the currently returned element using
213  * listDelNode(), but not to remove other elements.
214  *
215  * The function returns a pointer to the next element of the list,
216  * or NULL if there are no more elements, so the classical usage patter
217  * is:
218  *
219  * iter = listGetIterator(list,<direction>);
220  * while ((node = listNext(iter)) != NULL) {
221  *     doSomethingWith(listNodeValue(node));
222  * }
223  *
224  * */
225 listNode *listNext(listIter *iter)              //根据listIter获取当前迭代器指向的listNode,虽然名字是next，
226                                                 //但是指向的其实是当前的node，这个从以上分配iter的函数看出来
227 {
228     listNode *current = iter->next;
229 
230     if (current != NULL) {
231         if (iter->direction == AL_START_HEAD)
232             iter->next = current->next;
233         else
234             iter->next = current->prev;
235     }
236     return current;
237 }
238 
239 /* Duplicate the whole list. On out of memory NULL is returned.
240  * On success a copy of the original list is returned.
241  *
242  * The 'Dup' method set with listSetDupMethod() function is used
243  * to copy the node value. Otherwise the same pointer value of
244  * the original node is used as value of the copied node.
245  *
246  * The original list both on success or error is never modified. */
247 list *listDup(list *orig)
248 {
249     list *copy;
250     listIter *iter;
251     listNode *node;
252 
253     if ((copy = listCreate()) == NULL)                          //创建copy的list
254         return NULL;
255     copy->dup = orig->dup;                                      //相关函数指针赋值
256     copy->free = orig->free;
257     copy->match = orig->match;
258     iter = listGetIterator(orig, AL_START_HEAD);                //创建一个从头开始遍历的迭代器
259     while((node = listNext(iter)) != NULL) {                    //利用迭代器，逐步遍历list
260         void *value;
261 
262         if (copy->dup) {                                        //如果设置了dup复制函数，则通过该函数复制value
263             value = copy->dup(node->value);
264             if (value == NULL) {
265                 listRelease(copy);                              //只要是复制失败都要清理list和iter
266                 listReleaseIterator(iter);
267                 return NULL;
268             }
269         } else
270             value = node->value;                                //没指定dup函数直接赋值value
271         if (listAddNodeTail(copy, value) == NULL) {       //从尾端开始插入listNode，这样能保证新的list和原来的一样排序   
272             listRelease(copy);
273             listReleaseIterator(iter);
274             return NULL;
275         }
276     }
277     listReleaseIterator(iter);
278     return copy;
279 }
280 
281 /* Search the list for a node matching a given key.
282  * The match is performed using the 'match' method
283  * set with listSetMatchMethod(). If no 'match' method
284  * is set, the 'value' pointer of every node is directly
285  * compared with the 'key' pointer.
286  *
287  * On success the first matching node pointer is returned
288  * (search starts from head). If no matching node exists
289  * NULL is returned. */
290 listNode *listSearchKey(list *list, void *key)
291 {
292     listIter *iter;
293     listNode *node;                         //通过迭代器遍历list，逐个比较，算法复杂度O(n)
294 
295     iter = listGetIterator(list, AL_START_HEAD);
296     while((node = listNext(iter)) != NULL) {
297         if (list->match) {
298             if (list->match(node->value, key)) {
299                 listReleaseIterator(iter);
300                 return node;
301             }
302         } else {
303             if (key == node->value) {
304                 listReleaseIterator(iter);
305                 return node;
306             }
307         }
308     }
309     listReleaseIterator(iter);
310     return NULL;
311 }
312 
313 /* Return the element at the specified zero-based index
314  * where 0 is the head, 1 is the element next to head
315  * and so on. Negative integers are used in order to count
316  * from the tail, -1 is the last element, -2 the penultimate
317  * and so on. If the index is out of range NULL is returned. */
318 listNode *listIndex(list *list, long index) {
319     listNode *n;                    //负数的index，仅仅是表示从tail开始计数，就是说“负号”是表示方向是从tail开始
320 
321     if (index < 0) {
322         index = (-index)-1;
323         n = list->tail;
324         while(index-- && n) n = n->prev;
325     } else {
326         n = list->head;
327         while(index-- && n) n = n->next;
328     }
329     return n;
330 }
331 
332 /* Rotate the list removing the tail node and inserting it to the head. */
333 void listRotate(list *list) {                   //Rotate the list仅仅是将尾巴节点移动到头？
334     listNode *tail = list->tail;
335 
336     if (listLength(list) <= 1) return;
337 
338     /* Detach current tail */
339     list->tail = tail->prev;
340     list->tail->next = NULL;
341     /* Move it as head */
342     list->head->prev = tail;
343     tail->prev = NULL;
344     tail->next = list->head;
345     list->head = tail;
346 }
```

版权声明：本文为博主原创文章，未经博主允许不得转载。
