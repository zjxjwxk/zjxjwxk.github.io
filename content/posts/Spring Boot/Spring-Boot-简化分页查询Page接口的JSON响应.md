---
title: Spring Boot 简化分页查询Page接口的JSON响应
date: 2019-02-02 17:15:00
tags: 
- Java
- Spring Boot
categories: 
- Spring Boot
---



### 多余的分页信息

Spring Data 的 Page 接口进行数据分页返回了很多很多垃圾信息：

```json
{
    "status": 0,
    "data": {
        "content": [
            {
                "goodsId": 1,
                "producerId": 1,
                "categoryId": 4,
                "name": "耐克sb滑板鞋",
                "price": 500,
                "stock": 500,
                "status": "正常",
                "createTime": "2019-02-02T03:48:33.000+0000",
                "updateTime": null
            },
            {
                "goodsId": 2,
                "producerId": 1,
                "categoryId": 4,
                "name": "阿迪达斯小白鞋",
                "price": 500,
                "stock": 300,
                "status": "正常",
                "createTime": "2019-02-02T03:48:59.000+0000",
                "updateTime": null
            }
        ],
        "pageable": {
            "sort": {
                "sorted": false,
                "unsorted": true,
                "empty": true
            },
            "offset": 0,
            "pageSize": 10,
            "pageNumber": 0,
            "paged": true,
            "unpaged": false
        },
        "last": true,
        "totalPages": 1,
        "totalElements": 2,
        "size": 10,
        "number": 0,
        "numberOfElements": 2,
        "sort": {
            "sorted": false,
            "unsorted": true,
            "empty": true
        },
        "first": true,
        "empty": false
    }
}
```

### 实际需要的分页信息

实际上我只需要下面这些分页信息：

```json
"totalPages": 3,
"totalElements": 20,
"pageNumber": 2,
"numberOfElements": 7
```

### 创建 PageChunk 类进行封装

我的 GoodsRepository 接口为：

```java
Page<Goods> findByCategoryId(Integer categoryId, Pageable pageable);
```

我们看到返回了很多多余的，重复的数据。为了解决这个问题，创建一个Pagination DTO(`PageChunk`)来包装一下分页数据。这个类的作用很简单就是包含每一次分页的内容和分页元数据（**总页数**，**搜索结果总数**，**当前页号**， **当前页包含多少项**）。

```java
import lombok.Data;

import java.util.List;

/**
 * @author zjxjwxk
 */
@Data
public class PageChunk<T> {

    private List<T> content;
    private int totalPages;
    private long totalElements;
    private int pageNumber;
    private int numberOfElements;
}

```

同时，我根据业务需求对商品信息进行了DTO的包装，（只需要商品id，名称和价格即可）。

```java
import lombok.Data;

/**
 * @author zjxjwxk
 */
@Data
public class GoodsDTO {

    private Integer goodsId;
    private String name;
    private Integer price;

}
```

### 用 PageChunk 替代 Page

编写以下 getPageChunk 方法用于将 Page<Goods> 对象转化为 PageChunk<GoodsDTO> 对象，仅仅提取 Page 对象中所需要的Content 和一些分页信息，将 PageChunk 对象封装并返回。编写 getGoodsDTO 方法用于简化商品信息。

```java
@Service("GoodsService")
public class GoodsServiceImpl implements GoodsService {

    private final GoodsRepository goodsRepository;

    @Autowired
    public GoodsServiceImpl(GoodsRepository goodsRepository) {
        this.goodsRepository = goodsRepository;
    }

    @Override
    public ServerResponse getList(String keyword, Integer categoryId,
                                               Integer pageNum, Integer pageSize,
                                               String orderBy) {
        PageRequest pageRequest = PageRequest.of(pageNum - 1, pageSize);
        Page<Goods> goodsPage = goodsRepository.findByCategoryId(categoryId, pageRequest);
        return ServerResponse.createBySuccess(getPageChunk(goodsPage));
    }

    private PageChunk<GoodsDTO> getPageChunk(Page<Goods> goodsPage) {
        PageChunk<GoodsDTO> pageChunk = new PageChunk<>();
        pageChunk.setContent(getGoodsDTO(goodsPage.getContent()));
        pageChunk.setTotalPages(goodsPage.getTotalPages());
        pageChunk.setTotalElements(goodsPage.getTotalElements());
        pageChunk.setPageNumber(goodsPage.getPageable().getPageNumber() + 1);
        pageChunk.setNumberOfElements(goodsPage.getNumberOfElements());
        return pageChunk;
    }

    private List<GoodsDTO> getGoodsDTO(List<Goods> goodsList) {
        List<GoodsDTO> goodsDTOList = new ArrayList<>();
        for (Goods goods :
                goodsList) {
            GoodsDTO goodsDTO = new GoodsDTO();
            goodsDTO.setGoodsId(goods.getGoodsId());
            goodsDTO.setName(goods.getName());
            goodsDTO.setPrice(goods.getPrice());

            goodsDTOList.add(goodsDTO);
        }
        return goodsDTOList;
    }
}
```

> 为了返回 JSON 数据，我自定义了一个 ServerResponse 服务响应对象，它的作用是让我们返回的 JSON 有一个通用的格式

### 简化后的结果

```json
{
    "status": 0,
    "data": {
        "content": [
            {
                "goodsId": 8,
                "name": "Clarks 男 Tri Spark生活休闲鞋26135668",
                "price": 459
            },
            {
                "goodsId": 9,
                "name": "Adidas三叶草 男 休闲跑步鞋 STAN SMITH M20324",
                "price": 415
            },
            {
                "goodsId": 10,
                "name": "Skechers 斯凯奇 SKECHERS SPORT系列 男 绑带运动鞋 999732-NVOR",
                "price": 355
            },
            {
                "goodsId": 11,
                "name": "Saucony 圣康尼 RSP 男 休闲跑步鞋 KINETA RELAY S2524465",
                "price": 479
            },
            {
                "goodsId": 12,
                "name": "Mustang 男 帆布鞋休闲运动 4058-305（亚马逊进口直采,德国品牌）",
                "price": 324
            },
            {
                "goodsId": 13,
                "name": "Clarks 男 Tri Spark生活休闲鞋26135655",
                "price": 123425
            },
            {
                "goodsId": 14,
                "name": "New Balance 574系列 男 休闲跑步鞋 ML574EG-D",
                "price": 324
            }
        ],
        "totalPages": 3,
        "totalElements": 20,
        "pageNumber": 2,
        "numberOfElements": 7
    }
}
```