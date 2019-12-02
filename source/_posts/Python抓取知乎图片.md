title: Python抓取知乎图片
date: 2016-03-14 23:06:25
tags:
---


最近在学python，写个简单的爬虫来练练手。

### 思路

为了爬取的速度，所以目的是写个多线程的爬虫来爬取知乎的图片。

大致思路是模拟用户登录，从首页进入，下载其中的所有图片，并存储首页中的链接，然后再递归的查询存储的链接，下载图片，如此循环。

代码中用到的数据结构：
> download_image_queue 队列，用来存储所有的链接。 会有多个下载图片的线程通过竞争从这个队列中pop出链接，然后下载其中的图片

> generate_url_queue 队列，用来通过链接来查找链接。从download_image_queue pop出来的链接，除了下载其中的图片外，还需要put到这个队列中，有一个单独的线程来从这个队列中pop出链接，来查找这个链接中的其他所有链接，并存入download_image_queue队列。

> queried_set Set集合，用于存储已经查询过的链接，避免陷入死循环

<!-- more -->

### 代码

```python
from bs4 import BeautifulSoup
import uuid
import queue
import threading
import requests
import time
import os


def get_image_name(href):
    href_list = href.split('/')
    return href_list.pop()


def rename_image(name):
    name_list = name.split('.')
    if name_list is not None and name_list.pop() not in image_suffix():
        name = str(uuid.uuid4()) + '.jpg'
    return name


def image_suffix():
    image_suffixes = ['jpg', 'jpeg', 'png', 'gif']
    return image_suffixes


def create_dir(download_dir):
    if not os.path.exists(download_dir):
        os.mkdir(download_dir)


class LearnSpider:
    def __init__(self, username, password):
        self.queried_set = set()
        self.generate_url_queue = queue.Queue()
        self.download_image_queue = queue.Queue()
        self.session = self.login_zhihu(username, password)

    @staticmethod
    def login_zhihu(username, password):
        data = {
            "email": username,
            "password": password,
            "remember_me": "true",
        }
        login_url = "https://www.zhihu.com/login/email"
        session = requests.session()
        response = session.post(url=login_url, data=data)
        print(response.json())
        return session

    def parse_html(self, path):
        html = self.session.get(path).text
        soup = BeautifulSoup(html, 'html.parser')
        return soup

    def generate_path(self, path):
        soup = self.parse_html(path)
        link_list = soup.find_all("a")
        for link in link_list:
            href_value = link.get('href')
            if href_value is not None and href_value.startswith('http'):
                if href_value not in self.queried_set:
                    self.queried_set.add(href_value)
                    self.download_image_queue.put(href_value)

    def download_images(self, path, download_dir):
        self.generate_url_queue.put(path)
        soup = self.parse_html(path)
        a_list = soup.find_all('img')
        for link in a_list:
            src_value = link.get('src')
            if src_value is not None and src_value.startswith('http'):
                print(threading.current_thread().name, " downloading ", src_value)
                name = rename_image(get_image_name(src_value))
                with open("{0}{1}{2}".format(download_dir, os.path.sep, name), 'wb') as outfile:
                    data = self.session.get(src_value).content
                    outfile.write(data)

    def generate_path_work(self):
        while True:
            if not self.generate_url_queue.empty():
                pop_path = self.generate_url_queue.get()
                self.generate_path(pop_path)
            else:
                time.sleep(1)
                print("generate queue is null")

    def download_image_work(self, download_path):
        while True:
            if not self.download_image_queue.empty():
                pop_path = self.download_image_queue.get()
                self.download_images(pop_path, download_path)
            else:
                time.sleep(1)
                print("download queue is null")

    def start_download(self, download_dir, download_thread_count=10):
        create_dir(download_dir)
        path = "https://www.zhihu.com"
        self.download_image_queue.put(path)
        generate_thread = threading.Thread(target=self.generate_path_work, name="generate_path_thread")
        for i in range(1, download_thread_count + 1):
            threading.Thread(target=self.download_image_work, args=(download_dir,), name="thread{0}".format(i)).start()
        generate_thread.start()

if __name__ == "__main__":
    login_username = "***"
    login_password = "***"
    spider = LearnSpider(login_username, login_password)
    spider.start_download("e:\\down_images", download_thread_count=50)
```