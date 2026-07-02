---
title: 浅析Dockerhub API：如何优雅地从dockerhub偷rootfs镜像
date: 2024-10-02 09:34:55
tags:
  - Linux
  - Container
  - Docker
top_img: /img/dockerhub.jpg
cover: /img/dockerhub.jpg
---
# 成品：
https://github.com/Moe-hacker/docker_image_puller
# 前言：
八月初的时候，咱无聊去扒了下dockerhub的接口，想通过网络请求直接从dockerhub偷镜像。
然后写完才想起来dockkerhub在国内是被墙的，似乎这么一个功能用处也不大。。。。。
然后咱就去旅游了，连项目Readme都没写（逃）。
至于现在为啥写这篇文章，因为上课摸鱼。。。
# 前置函数：
```python
def panic(message):
    print("\033[31m", end="")
    print(message)
    print("\033[0m", end="")
    exit(1)


# Get the architecture of host.
def get_host_arch():
    # TODO: add more architecture support.
    arch = platform.machine()
    if arch == "x86_64":
        arch = "amd64"
    elif arch == "AMD64":
        arch = "amd64"
    elif arch == "aarch64":
        arch = "arm64"
    return arch
```
# 关于image字段：
除了docker search的实现之外，其他如果不带repo前缀的默认需要在image字段加入library/前缀，比如ubuntu镜像的image字段应该是library/ubuntu。
# 镜像查找：
首先是镜像的查找，也就是docker search的实现。
这个接口十分简单，需要两个参数：要查找的字段(image)和获取内容的条数(page_size)。
```python
    response = requests.get("https://hub.docker.com/v2/search/repositories/?page_size=" + str(page_size) + "&query=" + image)
```
设image="ubuntu", page_size=1, 在response.text中你会获得一段这样的json：
```json
{
  "count": 10000,
  "next": "https://hub.docker.com/v2/search/repositories/?page=2&page_size=1&query=ubuntu",
  "previous": "",
  "results": [
    {
      "repo_name": "ubuntu",
      "short_description": "Ubuntu is a Debian-based Linux operating system based on free software.",
      "star_count": 17303,
      "pull_count": 9419966493,
      "repo_owner": "",
      "is_automated": false,
      "is_official": true
    }
  ]
}
```
所以我们直接解析json输出就好了。
```python
    response = json.loads(response.text)
    for i in range(len(response['results'])):
        if response['results'][i]['is_official']:
            print("\033[33m" + response['results'][i]['repo_name'] + "\033[34m [official]")
        else:
            print("\033[33m" + response['results'][i]['repo_name'])
        if response['results'][i]['short_description'] == "":
            print("  \033[32mNo description found\033[0m")
        else:
            print("  \033[32m" + response['results'][i]['short_description'] + "\033[0m")
```
输出如下：
```text
ubuntu [official]
  Ubuntu is a Debian-based Linux operating system based on free software.
```
然后是tag的查找，这个API和上面差不多, 给出镜像名image和数据条数page_size：
```python
    response = requests.get("https://hub.docker.com/v2/repositories/" + image + "/tags/?page_size=" + str(page_size))
```
设image="library/ubuntu", page_size=1, 在response.text中你会获得一段这样的json：
```json
{
    "count": 644,
    "next": "https://hub.docker.com/v2/repositories/ubuntu/tags/?page=2\u0026page_size=1",
    "previous": null,
    "results": [
        {
            "creator": 621950,
            "id": 3888826,
            "images": [
                {
                    "architecture": "amd64",
                    "features": "",
                    "variant": null,
                    "digest": "sha256:164c01664783caf799e9026babceea81bfa6341686f7296da93769ceab803583",
                    "os": "linux",
                    "os_features": "",
                    "os_version": null,
                    "size": 34157024,
                    "status": "active",
                    "last_pulled": "2024-10-02T23:51:41.525479Z",
                    "last_pushed": "2024-10-02T01:26:16.419417Z"
                },
                {
                    "architecture": "arm",
                    "features": "",
                    "variant": "v7",
                    "digest": "sha256:1c274f6fa5acd46af265b71438c44eaee8f8db6a5d901e972920e311cfc69326",
                    "os": "linux",
                    "os_features": "",
                    "os_version": null,
                    "size": 33319358,
                    "status": "active",
                    "last_pulled": "2024-10-02T23:21:46.615696Z",
                    "last_pushed": "2024-10-02T01:26:16.551432Z"
                },
                {
                    "architecture": "arm64",
                    "features": "",
                    "variant": "v8",
                    "digest": "sha256:f64ecfbbd782f5b1827c225e9ffbb211d5c1898cc21278aa654bc05621943611",
                    "os": "linux",
                    "os_features": "",
                    "os_version": null,
                    "size": 33752693,
                    "status": "active",
                    "last_pulled": "2024-10-02T23:50:16.569001Z",
                    "last_pushed": "2024-10-02T01:01:14.99007Z"
                },
                {
                    "architecture": "ppc64le",
                    "features": "",
                    "variant": null,
                    "digest": "sha256:cce4f13e55e73c568d263542667529c3a4f86ce95a0c7356f272bc8f03506b7d",
                    "os": "linux",
                    "os_features": "",
                    "os_version": null,
                    "size": 38594353,
                    "status": "active",
                    "last_pulled": "2024-10-02T23:21:46.985196Z",
                    "last_pushed": "2024-10-02T01:26:16.672456Z"
                },
                {
                    "architecture": "riscv64",
                    "features": "",
                    "variant": null,
                    "digest": "sha256:15b1b51fd69ef84b63e5e6fa7182fa414edd4d245a0a921f383a6bb6724d7425",
                    "os": "linux",
                    "os_features": "",
                    "os_version": null,
                    "size": 37487731,
                    "status": "active",
                    "last_pulled": "2024-10-02T23:21:47.353319Z",
                    "last_pushed": "2024-10-02T01:26:35.383054Z"
                },
                {
                    "architecture": "s390x",
                    "features": "",
                    "variant": null,
                    "digest": "sha256:891d888358d9f17b68429662b28d0e0db1bc019bf183abac1109ff8c0de24df7",
                    "os": "linux",
                    "os_features": "",
                    "os_version": null,
                    "size": 33947344,
                    "status": "active",
                    "last_pulled": "2024-10-02T23:26:58.16971Z",
                    "last_pushed": "2024-10-02T01:01:15.107453Z"
                }
            ],
            "last_updated": "2024-10-02T01:50:27.90536Z",
            "last_updater": 1156886,
            "last_updater_username": "doijanky",
            "name": "devel",
            "repository": 130,
            "full_size": 34157024,
            "v2": true,
            "tag_status": "active",
            "tag_last_pulled": "2024-10-02T23:51:41.525479Z",
            "tag_last_pushed": "2024-10-02T01:50:27.90536Z",
            "media_type": "application/vnd.oci.image.index.v1+json",
            "content_type": "image",
            "digest": "sha256:c62f1babc85f8756f395e6aabda682acd7c58a1b0c3bea250713cd0184a93efa"
        }
    ]
}
```
这里解析复杂一点，需要results:images:architecture和主机架构一致：
```python
    if response.status_code != 200:
        panic("Search failed!")
    response = json.loads(response.text)
    found = False
    for i in range(len(response['results'])):
        for j in range(len(response['results'][i]['images'])):
            if response['results'][i]['images'][j]['architecture'] == get_host_arch():
                found = True
                print("\033[32m[" + image + "] " + response['results'][i]['name'] + "\033[0m")
                break
    if found != True:
        panic("No result!")
```
输出如下：
```text
[ubuntu] latest
```
# rootfs获取：
这里超级坑，步骤有点多我们慢慢来：
获取token：
虽然不知道为什么，但我们在获取docker镜像前需要先申请一个token，服务器在后续请求时需要验证你的token。
我们只需要提供镜像名image，就是上面docker search出来的那个名字
```python
    response = requests.get(
        "https://auth.docker.io/token?service=registry.docker.io&scope=repository%3A" + image +
        "%3Apull")
```
我们会获得这样一段json：
```json
{
    "token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsIng1YyI6WyJNSUlFRmpDQ0F2NmdBd0lCQWdJVUlvQW42a0k2MUs3bTAwNVhqcXVpKzRDTzVUb3dEUVlKS29aSWh2Y05BUUVMQlFBd2dZWXhDekFKQmdOVkJBWVRBbFZUTVJNd0VRWURWUVFJRXdwRFlXeHBabTl5Ym1saE1SSXdFQVlEVlFRSEV3bFFZV3h2SUVGc2RHOHhGVEFUQmdOVkJBb1RERVJ2WTJ0bGNpd2dTVzVqTGpFVU1CSUdBMVVFQ3hNTFJXNW5hVzVsWlhKcGJtY3hJVEFmQmdOVkJBTVRHRVJ2WTJ0bGNpd2dTVzVqTGlCRmJtY2dVbTl2ZENCRFFUQWVGdzB5TkRBNU1qUXlNalUxTURCYUZ3MHlOVEE1TWpReU1qVTFNREJhTUlHRk1Rc3dDUVlEVlFRR0V3SlZVekVUTUJFR0ExVUVDQk1LUTJGc2FXWnZjbTVwWVRFU01CQUdBMVVFQnhNSlVHRnNieUJCYkhSdk1SVXdFd1lEVlFRS0V3eEViMk5yWlhJc0lFbHVZeTR4RkRBU0JnTlZCQXNUQzBWdVoybHVaV1Z5YVc1bk1TQXdIZ1lEVlFRREV4ZEViMk5yWlhJc0lFbHVZeTRnUlc1bklFcFhWQ0JEUVRDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTGRWRDVxNlJudkdETUxPVysrR1MxWENwR2FRRHd0V3FIS2tLYlM5cVlJMXdCallKWEJ6U2MweTBJK0swU0lVd2pqNGJJT3ZpNXNyOGhJajdReGhrY1ppTlU1OEE5NW5BeGVFS3lMaU9QU0tZK3Y5VnZadmNNT2NwVW1xZ1BxWkhoeTVuMW8xbGxmek92dTd5SDc4a1FyT0lTMTZ3RFVVZm8yRkxPaERDaElsbCtYa2VlbFB6c0tiRWo3ZGJqdXV6RGxIODlWaE4yenNWNFV3c244UVpGVTB4V00wb3R2d0lQN3k0UDZGWDBuUFJuTVQyMlRIajVIWVJ3SUFVdm1FN0l3YlZVQ2wvM1hPaGhwbGNJZFQxREZGOUJUMHJOUC93ZTBWMklId1RHdVdTVENWb3M2b3R5ekk3a1hEdGZzeWRjU2Q5TklpSXZITHFYamJPVGtidWVjQ0F3RUFBYU43TUhrd0RnWURWUjBQQVFIL0JBUURBZ0dtTUJNR0ExVWRKUVFNTUFvR0NDc0dBUVVGQndNQk1CSUdBMVVkRXdFQi93UUlNQVlCQWY4Q0FRQXdIUVlEVlIwT0JCWUVGSmZWdXV4Vko3UXh1amlMNExZajFjQjEzbWhjTUI4R0ExVWRJd1FZTUJhQUZDNjBVUE5lQmtvZ1kyMnRYUGNCTUhGdkczQ3NNQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUJBUUFiQkdlZHZVVzhOVWp1VXJWbDlrWmMybDRDbjhJbDFzeVBVTDNYVXdQSHprcy9iUFJ4S1loeFlIODdOb1NwdDZJT3ZPS0k3ZCthQmoyM1lldTdDWGltTWxMUWl4UGhwQ0J0dC92Vmx1UXNJbVZXZXBJWCtraENienNGemtNbUNpbHo1OXVxOURiaGg3VUw1NjQxUjdFQ2pZc0h0Y2RKeURXRWFqMXFEVFoyOUUwY1UvWmhmbmsrVFVOTExkNjYxNldCREQ3TDlSNkgzK3VkRXBRRDFlcXYzU1YwczY3R2ZVT3l0RXhzMVRja3U4aUJCdnJLbnhZa3BZMVZDbW5UMUxSaFo4K283YU94YjR4ZHByMFR6ZnBqN3BidEhWQnQwSGNUUlpIdG54MkhCN3pzWXRFZUl3eVE3bGhhMVJ4eDJNQmR0R2tBREFLUk9RRnpmMEJubm91TSJdfQ.eyJhY2Nlc3MiOlt7ImFjdGlvbnMiOlsicHVsbCJdLCJuYW1lIjoibGlicmFyeS91YnVudHUiLCJwYXJhbWV0ZXJzIjp7InB1bGxfbGltaXQiOiIxMDAiLCJwdWxsX2xpbWl0X2ludGVydmFsIjoiMjE2MDAifSwidHlwZSI6InJlcG9zaXRvcnkifV0sImF1ZCI6InJlZ2lzdHJ5LmRvY2tlci5pbyIsImV4cCI6MTcyNzkxODE2OSwiaWF0IjoxNzI3OTE3ODY5LCJpc3MiOiJhdXRoLmRvY2tlci5pbyIsImp0aSI6ImRja3JfanRpX1o1czVaNWJhRFlHUEZ0eGdDZGFkZGRZQ29WRT0iLCJuYmYiOjE3Mjc5MTc1NjksInN1YiI6IiJ9.CBSS2U9kYgWqMk0fQZBq5FD_U4qPhnqzdidN5PoPrcDX7M-6UxgoCbP92gDtEqJ4FFPIUMTdthU4jt_6vB-2W46pQk4xBk2vQ-ZpUHk5JbSW4sB8HM-LdzHfp62L9Ec3mH2BkoFhswYfbtDlR_KpNxrGEdi-4AomxDjzQLOdeBl4ww5li3dvpVgGQxM7kjHtq2fMJ3E33U5ZM96ZTZK2imtrTSNpOenZq4rm96XqWKE7wjcH1yWFu6qbl9fnCR43LcTWGCtVXMpdndX5ouAR4qtcMbftPL_jTvKNf-z04OUyigjyoLtCK7iahF0stkkyuWMjE_-O8GfkhlPR8x-qrQ",
    "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsIng1YyI6WyJNSUlFRmpDQ0F2NmdBd0lCQWdJVUlvQW42a0k2MUs3bTAwNVhqcXVpKzRDTzVUb3dEUVlKS29aSWh2Y05BUUVMQlFBd2dZWXhDekFKQmdOVkJBWVRBbFZUTVJNd0VRWURWUVFJRXdwRFlXeHBabTl5Ym1saE1SSXdFQVlEVlFRSEV3bFFZV3h2SUVGc2RHOHhGVEFUQmdOVkJBb1RERVJ2WTJ0bGNpd2dTVzVqTGpFVU1CSUdBMVVFQ3hNTFJXNW5hVzVsWlhKcGJtY3hJVEFmQmdOVkJBTVRHRVJ2WTJ0bGNpd2dTVzVqTGlCRmJtY2dVbTl2ZENCRFFUQWVGdzB5TkRBNU1qUXlNalUxTURCYUZ3MHlOVEE1TWpReU1qVTFNREJhTUlHRk1Rc3dDUVlEVlFRR0V3SlZVekVUTUJFR0ExVUVDQk1LUTJGc2FXWnZjbTVwWVRFU01CQUdBMVVFQnhNSlVHRnNieUJCYkhSdk1SVXdFd1lEVlFRS0V3eEViMk5yWlhJc0lFbHVZeTR4RkRBU0JnTlZCQXNUQzBWdVoybHVaV1Z5YVc1bk1TQXdIZ1lEVlFRREV4ZEViMk5yWlhJc0lFbHVZeTRnUlc1bklFcFhWQ0JEUVRDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTGRWRDVxNlJudkdETUxPVysrR1MxWENwR2FRRHd0V3FIS2tLYlM5cVlJMXdCallKWEJ6U2MweTBJK0swU0lVd2pqNGJJT3ZpNXNyOGhJajdReGhrY1ppTlU1OEE5NW5BeGVFS3lMaU9QU0tZK3Y5VnZadmNNT2NwVW1xZ1BxWkhoeTVuMW8xbGxmek92dTd5SDc4a1FyT0lTMTZ3RFVVZm8yRkxPaERDaElsbCtYa2VlbFB6c0tiRWo3ZGJqdXV6RGxIODlWaE4yenNWNFV3c244UVpGVTB4V00wb3R2d0lQN3k0UDZGWDBuUFJuTVQyMlRIajVIWVJ3SUFVdm1FN0l3YlZVQ2wvM1hPaGhwbGNJZFQxREZGOUJUMHJOUC93ZTBWMklId1RHdVdTVENWb3M2b3R5ekk3a1hEdGZzeWRjU2Q5TklpSXZITHFYamJPVGtidWVjQ0F3RUFBYU43TUhrd0RnWURWUjBQQVFIL0JBUURBZ0dtTUJNR0ExVWRKUVFNTUFvR0NDc0dBUVVGQndNQk1CSUdBMVVkRXdFQi93UUlNQVlCQWY4Q0FRQXdIUVlEVlIwT0JCWUVGSmZWdXV4Vko3UXh1amlMNExZajFjQjEzbWhjTUI4R0ExVWRJd1FZTUJhQUZDNjBVUE5lQmtvZ1kyMnRYUGNCTUhGdkczQ3NNQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUJBUUFiQkdlZHZVVzhOVWp1VXJWbDlrWmMybDRDbjhJbDFzeVBVTDNYVXdQSHprcy9iUFJ4S1loeFlIODdOb1NwdDZJT3ZPS0k3ZCthQmoyM1lldTdDWGltTWxMUWl4UGhwQ0J0dC92Vmx1UXNJbVZXZXBJWCtraENienNGemtNbUNpbHo1OXVxOURiaGg3VUw1NjQxUjdFQ2pZc0h0Y2RKeURXRWFqMXFEVFoyOUUwY1UvWmhmbmsrVFVOTExkNjYxNldCREQ3TDlSNkgzK3VkRXBRRDFlcXYzU1YwczY3R2ZVT3l0RXhzMVRja3U4aUJCdnJLbnhZa3BZMVZDbW5UMUxSaFo4K283YU94YjR4ZHByMFR6ZnBqN3BidEhWQnQwSGNUUlpIdG54MkhCN3pzWXRFZUl3eVE3bGhhMVJ4eDJNQmR0R2tBREFLUk9RRnpmMEJubm91TSJdfQ.eyJhY2Nlc3MiOlt7ImFjdGlvbnMiOlsicHVsbCJdLCJuYW1lIjoibGlicmFyeS91YnVudHUiLCJwYXJhbWV0ZXJzIjp7InB1bGxfbGltaXQiOiIxMDAiLCJwdWxsX2xpbWl0X2ludGVydmFsIjoiMjE2MDAifSwidHlwZSI6InJlcG9zaXRvcnkifV0sImF1ZCI6InJlZ2lzdHJ5LmRvY2tlci5pbyIsImV4cCI6MTcyNzkxODE2OSwiaWF0IjoxNzI3OTE3ODY5LCJpc3MiOiJhdXRoLmRvY2tlci5pbyIsImp0aSI6ImRja3JfanRpX1o1czVaNWJhRFlHUEZ0eGdDZGFkZGRZQ29WRT0iLCJuYmYiOjE3Mjc5MTc1NjksInN1YiI6IiJ9.CBSS2U9kYgWqMk0fQZBq5FD_U4qPhnqzdidN5PoPrcDX7M-6UxgoCbP92gDtEqJ4FFPIUMTdthU4jt_6vB-2W46pQk4xBk2vQ-ZpUHk5JbSW4sB8HM-LdzHfp62L9Ec3mH2BkoFhswYfbtDlR_KpNxrGEdi-4AomxDjzQLOdeBl4ww5li3dvpVgGQxM7kjHtq2fMJ3E33U5ZM96ZTZK2imtrTSNpOenZq4rm96XqWKE7wjcH1yWFu6qbl9fnCR43LcTWGCtVXMpdndX5ouAR4qtcMbftPL_jTvKNf-z04OUyigjyoLtCK7iahF0stkkyuWMjE_-O8GfkhlPR8x-qrQ",
    "expires_in": 300,
    "issued_at": "2024-10-03T01:11:09.171382002Z"
}
```
我们只需要token就行了：
```python
    response = json.loads(response.text)
    if response.__contains__('token') == False:
        panic("Could not get token!")
    return response['token']
```
然后我们开始获取镜像。
首先获得镜像image标签tag对应的信息,这一步需要token：
```python
    response = requests.get("https://registry-1.docker.io/v2/" + image + "/manifests/" + tag,
                            headers={
                                "Authorization": "Bearer " + token,
                                "Accept": "application/vnd.docker.distribution.manifest.list.v2+json"
                            })
```
image="library/ubuntu",tag="latest"你将获得如下json：
```json
{
    "manifests": [
        {
            "digest": "sha256:74f92a6b3589aa5cac6028719aaac83de4037bad4371ae79ba362834389035aa",
            "mediaType": "application/vnd.oci.image.manifest.v1+json",
            "platform": {
                "architecture": "amd64",
                "os": "linux"
            },
            "size": 424
        },
        {
            "digest": "sha256:4192b2c3b4311ccb41d838a0e33bd1e403afe72d77d424432259dd81fba88edb",
            "mediaType": "application/vnd.oci.image.manifest.v1+json",
            "platform": {
                "architecture": "arm",
                "os": "linux",
                "variant": "v7"
            },
            "size": 424
        },
        {
            "digest": "sha256:1d392dca9b7043b31d8214bb8c42e4371df17a6304597c0fb4fe5d2060186b19",
            "mediaType": "application/vnd.oci.image.manifest.v1+json",
            "platform": {
                "architecture": "arm64",
                "os": "linux",
                "variant": "v8"
            },
            "size": 424
        },
        {
            "digest": "sha256:294858195461fe9a982d4c046e6d59ed65032087d32a2aafa734d0e230f06470",
            "mediaType": "application/vnd.oci.image.manifest.v1+json",
            "platform": {
                "architecture": "ppc64le",
                "os": "linux"
            },
            "size": 424
        },
        {
            "digest": "sha256:08d0f41b17da54c7c5bc30c9275b9cfad79e59915138325ba846f8d7b254c0e2",
            "mediaType": "application/vnd.oci.image.manifest.v1+json",
            "platform": {
                "architecture": "riscv64",
                "os": "linux"
            },
            "size": 424
        },
        {
            "digest": "sha256:48459f9f4d0b5444d5a43078d844fd5caf05b31b7553ae79e26d7a18200b9460",
            "mediaType": "application/vnd.oci.image.manifest.v1+json",
            "platform": {
                "architecture": "s390x",
                "os": "linux"
            },
            "size": 424
        }
    ],
    "mediaType": "application/vnd.oci.image.index.v1+json",
    "schemaVersion": 2
}
```
解析获得和主机架构对应的digest:
```python
    if response.status_code != 200:
        panic("Get manifests failed!")
    manifests = json.loads(response.text)
    for i in range(len(manifests['manifests'])):
        if manifests['manifests'][i]['platform']['architecture'] == get_host_arch():
            return manifests['manifests'][i]['digest']
    return None
```
最后我们解析这个digest，获得最终要下载的文件, 需要image，digest和token：
```python
response = requests.get("https://registry-1.docker.io/v2/" + image + "/manifests/" + digest,
                            headers={
                                "Authorization": "Bearer " + token,
                                "Accept": "application/vnd.oci.image.manifest.v1+json"
                            })
```
你将获得：
```json
{
    "schemaVersion": 2,
    "mediaType": "application/vnd.oci.image.manifest.v1+json",
    "config": {
        "mediaType": "application/vnd.oci.image.config.v1+json",
        "size": 2314,
        "digest": "sha256:c22ec0081bf1bc159b97b27b55812319e99b710df960a16c2f2495dc7fc61c00"
    },
    "layers": [
        {
            "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
            "size": 28885430,
            "digest": "sha256:25a614108e8d9c23a53cb3193f34ba623efe45c81ccaaa2281092b512ef7e07e"
        }
    ]
}
```
获取layers字段的digest：
```python
    if response.status_code != 200:
        panic("Get blobs failed!")
    manifests = json.loads(response.text)
    ret = []
    for i in range(len(manifests['layers'])):
        ret.append(manifests['layers'][i]['digest'])
    return ret
```
最后下载下来解压：
需要image, blobs（上面的ret）, token, 保存路径savedir：
```python
    os.system("mkdir " + savedir)
    for i in range(len(blobs)):
        print("\033[33mpulling: \033[32m" + blobs[i][7:] + "\033[0m")
        os.system("curl -s -L -H \"Authorization: Bearer " + token + "\"" +
                  " https://registry-1.docker.io/v2/" + image + "/blobs/" + blobs[i] + " -o " + savedir +
                  "/" + "rootfs_" + blobs[i][7:])
    os.system("cd " + savedir + " && sudo tar -xvf ./* && rm rootfs_*")
```
最后，有些docker镜像启动命令并非默认，我们可以解析config来获得它：
```python
def get_cmd(image, digest, token):
    print("")
    response = requests.get("https://registry-1.docker.io/v2/" + image + "/manifests/" + digest,
                            headers={
                                "Authorization": "Bearer " + token,
                                "Accept": "application/vnd.oci.image.manifest.v1+json"
                            })
    if response.status_code != 200:
        panic("Get config failed!")
    manifests = json.loads(response.text)
    config = manifests['config']['digest']
    response = requests.get("https://registry-1.docker.io/v2/" + image + "/blobs/" + config,
                            headers={
                                "Authorization": "Bearer " + token,
                                "Accept": "application/vnd.oci.image.manifest.v1+json"
                            })
    if response.status_code != 200:
        panic("Get config failed!")
    response = json.loads(response.text)
    print("\033[32mDefault start command is:\033[33m")
    for i in range(len(response['config']['Cmd'])):
        print(response['config']['Cmd'][i], end="")
        print(" ", end="")
    print("")
    print("\033[32mRun this command in container to start its service (If need)\033[0m")
```
config的json长这样：
```
{

    "architecture": "arm64",
    "config": {

        "Hostname": "",
        "Domainname": "",
        "User": "",
        "AttachStdin": false,
        "AttachStdout": false,
        "AttachStderr": false,
        "Tty": false,
        "OpenStdin": false,
        "StdinOnce": false,
        "Env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],
        "Cmd": ["/bin/bash"],
        "Image": "sha256:cda645029b39f2e6252f29a2d69bfeccf8dc50d034823b5cdca06607308c0f9b",
        "Volumes": null,
        "WorkingDir": "",
        "Entrypoint": null,
        "OnBuild": null,
        "Labels": {
            "org.opencontainers.image.ref.name": "ubuntu", "org.opencontainers.image.version": "24.04"
        }
    }

    ,
    "container": "9b94be65d3716f5d746c4c316209f417d0595f6d2999fffdf66cf658e18a40ee",
    "container_config": {

        "Hostname": "9b94be65d371",
        "Domainname": "",
        "User": "",
        "AttachStdin": false,
        "AttachStdout": false,
        "AttachStderr": false,
        "Tty": false,
        "OpenStdin": false,
        "StdinOnce": false,
        "Env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],
        "Cmd": ["/bin/sh",
        "-c",
        "#(nop) ",
        "CMD [\"/bin/bash\"]"],
        "Image": "sha256:cda645029b39f2e6252f29a2d69bfeccf8dc50d034823b5cdca06607308c0f9b",
        "Volumes": null,
        "WorkingDir": "",
        "Entrypoint": null,
        "OnBuild": null,
        "Labels": {
            "org.opencontainers.image.ref.name": "ubuntu", "org.opencontainers.image.version": "24.04"
        }
    }

    ,
    "created": "2024-09-16T06:23:59.011017704Z",
    "docker_version": "24.0.7",
    "history": [ {
        "created": "2024-09-16T06:23:55.680007304Z", "created_by": "/bin/sh -c #(nop)  ARG RELEASE", "empty_layer": true
    }

    ,
    {
    "created": "2024-09-16T06:23:55.713531864Z", "created_by": "/bin/sh -c #(nop)  ARG LAUNCHPAD_BUILD_ARCH", "empty_layer": true
}

,
{
"created": "2024-09-16T06:23:55.756039708Z", "created_by": "/bin/sh -c #(nop)  LABEL org.opencontainers.image.ref.name=ubuntu", "empty_layer": true
}

,
{
"created": "2024-09-16T06:23:55.807967157Z", "created_by": "/bin/sh -c #(nop)  LABEL org.opencontainers.image.version=24.04", "empty_layer": true
}

,
{
"created": "2024-09-16T06:23:58.625802969Z", "created_by": "/bin/sh -c #(nop) ADD file:9d8a15341d364d2508ffca0657b4cb175a35d2edbdb0cf2476f7c4fff4f6c66a in / "
}

,
{
"created": "2024-09-16T06:23:59.011017704Z", "created_by": "/bin/sh -c #(nop)  CMD [\"/bin/bash\"]", "empty_layer": true
}

],
"os": "linux",
"rootfs": {
    "type": "layers", "diff_ids": ["sha256:c26ee3582dcbad8dc56066358653080b606b054382320eb0b869a2cb4ff1b98b"]
}

,
"variant": "v8"
}
```
# 结尾：
EOF