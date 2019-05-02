---
layout: post
title:  "MSA(마이크로서비스) BackEnd 구현"
date:   2019-05-02 21:00:59
author: Jm Park
categories: MSA
tags: msa
---

# MSA구현 방식  
MSA방식의 Back-End 구현은 DDD방식을 따른다.  
**Domain <-> Service <-> Store**  
이들의 관계와 이것을 연결하는 Lifecycle의 집합니다.

# DDD설계 구조
가장 상단의 **Domain**에는 Entity/Logic/Spec/Store/Event/Lifecycle/Proxy 패키지가 포함된다.  
Domain을 호출하는 하위에는 Service, Store가 존재한다.  
**Service** 는 실제 Logic이 있으며, RestAPI를 호출한다. Logic/Rest/Lifecycle/Listen/Bind 패키지가 포함된다.  
**Store** 는 실제 물리적 데이터를 변화시키는데 사용된다. Store(Jpo,Repository)/Lifecycle 패키지가 포함된다.
**Boot** 는 프로젝트 세팅과 Back-End의 시작점이다.  

## Domain 구조
### 1. Entity  
하나의 객체 덩어리이다.
entity 패키지에는 다양한 덩어리의 Class들을 생성한다.  
아래는 그 예시다.  

```{.java}
package item.domain.entity;

import java.util.UUID;
import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class Item {
 private static final long serialVersionUID = 1L;

 private String id;
 private String name;
 private int buyingPrice;
 private int price;

 public Item() {
  super();
  this.id = UUID.randomUUID().toString();
 }

 // NameValueList는 임의로 만든 것
 public void setValues(NameValueList nameValues) {
  for (NameValue nameValue : nameValues.getList()) {
   String name = nameValue.getName();
   String value = nameValue.getValue();

   switch (name) {  // setValue를 하는 역할이다.
   case "name":
    this.name = value;
    break;
   case "buyingPrice":
    this.buyingPrice = Integer.valueOf(value);
    break;
   case "price":
    this.price = Integer.valueOf(value);
    break;
   }
  }
 }

 public static Item fromJson(String json) {
  return JsonUtil.fromJson(json, Item.class);
 }

 public static Item getSample() {
  Item thisEntity = new Item();
  thisEntity.setBuyingPrice(1000);
  thisEntity.setPrice(900);
  thisEntity.setName("Productory");
  return thisEntity;
 }

 public static void main(String[] args) {
  System.out.println(getSample().toString());
 }

}
```

### 2. Spec  
Entity를 활용할 수 있도록 Entity를 조작하는 행위. 즉, 서비스에서 Logic을 수행하기 위한 행위를 정의(Interface)한다.    

```{.java}  
package item.domain.spec;

import item.domain.entity.Item;

public interface ItemService {

  public String register(Item item);
  public Item find(String id);
  public void modify(String id, NameValueList nameValue);
  public void remove(String id);

}
```

### 3. Store
데이터의 물리적인 접근(DB접근)을 위한 행위를 정의(Interface)한다.  

```{.java}  
package item.domain.store;

import item.domain.entity.Item;

public interface ItemStore {

  public void create(Item item); // 생성
  public Item retrieve(String id); // 조회
  public void update(Item item); // 수정
  public void delete(String id); // 삭제
  public void delete(Item item);
}
```

### 4. Lifecycle  
Entity 등 Domain에 접근하기 위한 통로의 역할을 한다.  

```{.java}  
package item.domain.lifecycle;

import item.domain.spec.ItemService;

public interface ServiceLifecycle {
  ItemService requestItemService();
}
```

### 5. Logic  
Logic은 Spec에서 정의(interface로 선언된)한 메소드의 기능을 구체화 시킨다.  
여기서 구체화 작업이란, Sotre에 정의(interface로 선언된)한 메소드를 활용하여 DB의 데이터 변화를 일으키는 행위를 포함하여 메소드를 구현한다.  
즉, DATA의 물리적인 변화를 큰 흐름으로 구현해간다.  

```{.java} 
package item.domain.logic;

import item.domain.entity.Item;
import item.domain.event.ItemEvent;
import item.domain.spec.ItemService;
import item.domain.store.ItemStore;
import item.domain.lifecycle.StoreLifecycle;

public class ItemLogic implements ItemService {

 ItemStore itemStore;

 public ItemLogic(StoreLifecycle storeLifecycle) {
  itemStore = storeLifecycle.requestItemStore();
 }

 @Override
 public String register(Item item) {
  // 생성
  this.itemStore.create(item);
  return item.getId();
 }

 @Override
 public Item find(String id) {
  // 조회
  return this.itemStore.retrieve(id);
 }

 @Override
 public void modify(String id, NameValueList nameValues) {
  // 수정
  Item item = this.itemStore.retrieve(id);
  item.setValues(nameValues);
  this.itemStore.update(item);
 }

 @Override
 public void remove(String id) {
  // 삭제
  Item item = this.itemStore.retrieve(id);
  this.itemStore.delete(item);
 }

}
```

## Service 구조  
### 1. Lifecycle   
Domain 영역의 Service를 활용하기 위해 ServiceLifecycler를 생성해 도메인과 연결하는 징검다리 역할이다.  

```{.java}
package item.lifecycle;

import org.springframework.stereotype.Component;

import item.domain.lifecycle.ServiceLifecycle;
import item.domain.spec.ItemService;

@Component
public class ServiceLifecycler implements ServiceLifecycle {
    
 private final ItemService itemService;
 
 public ServiceLifecycler(ItemService itemService){
  this.itemService = itemService;
 }
 
 public ItemService requestItemService(){
  return this.itemService;
 }
}
```

### 2. Logic  
Domain의 Logic 부분을 상속받아 Service에서 활용할 수 있게 된다.  
여기서 Domain의 영역에서 정의한 것 외에 기능을 상세히하여 추가하는 작업을 할 수 있다.   

```{.java}
package item.logic;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

import item.domain.lifecycle.StoreLifecycle;
import item.domain.logic.ItemLogic;

@Service
@Transactional(propagation = Propagation.REQUIRED)
public class ItemSpringLogic extends ItemLogic {

    public ItemSpringLogic(StoreLifecycle storeLifecycle) {
        super(storeLifecycle);
        // TODO Auto-generated constructor stub
    }

}
```

### 3. rest  
정의한 기능들을 사용하기 위해 RESTAPI로 정의해주는 곳이다.  
사용할 메소드와 url을 정의해준다.  
이곳에서 메소드를 응용하는 역할을 할 수 있다.  

```{.java}
package item.rest;

import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import item.domain.entity.Item;
import item.domain.lifecycle.ServiceLifecycle;
import item.domain.spec.ItemService;

@RestController
@RequestMapping(value = "/item")
public class ItemResource {

 public final ItemService itemService;

 public ItemResource(ServiceLifecycle serviceLifecycle) {
  this.itemService = serviceLifecycle.requestItemService();
 }

 @PostMapping(value = "/")
 public String register(@RequestBody Item item) {
  // 생성
  return itemService.register(item);
 }

 @GetMapping(value = "/{id}")
 public Item find(@PathVariable(value = "id") String id) {
  // 조회
  return itemService.find(id);
 }

 @PutMapping(value = "/{id}")
 public void modify(@PathVariable(value = "id") String id, @RequestBody NameValueList nameValue) {
  // 수정
  itemService.modify(id, nameValue);
 }

 @DeleteMapping(value = "/{id}")
 public void remove(@PathVariable(value = "id") String id) {
  // 삭제
  itemService.remove(id);
 }

}
```

## Store  
## 1. store - jpo   
물리적인 테이블과 1:1 맵핑이 이루어지는 곳이다.  
이 맵핑을 통해 실제 DB 항목들에 접근할 수 있다.  

```{.java}
package item.store.jpo;

import java.util.List;
import java.util.stream.Collectors;
import java.util.stream.StreamSupport;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.Id;

import org.springframework.beans.BeanUtils;

import item.domain.entity.Item;

import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
@Entity
public class ItemJpo {

 @Id
 @Column(name = "ID")
 private String id;
 @Column(name = "NM")
 private String name;
 @Column(name = "B_PRC")
 private int buyingPrice;
 @Column(name = "PRC")
 private int price;

 public ItemJpo(Item entity) {
  BeanUtils.copyProperties(entity, this);
 }

 public Item toDomain() {
  Item entity = new Item();
  BeanUtils.copyProperties(this, entity);
  return entity;
 }

 public static List<Item> toDomains(Iterable<ItemJpo> jpos) {
  /* @formatter:off */
  return StreamSupport.stream(jpos.spliterator(), false).map(ItemJpo::toDomain).collect(Collectors.toList());
  /* @formatter:on */
 }

}
```

### 2. stroe - repository 
JPA는 SpringBoot에서 제공하는, SQL을 사용하지 않고 메소드를 통해 SQL과 유사한 성능을 보이는 메소드 라이브러리이다.  
이곳에 사용할 SQL메소드들을 정의하여 활용할 수 있다.  

[JPA 설명 링크](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
```{.java}
package item.store.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import item.store.jpo.ItemJpo;

public interface ItemRepository extends JpaRepository<ItemJpo, String> {

}
```

### 3. store  
store아래 jpo/repository 패키지들과 달리 store 패키지에 바로 연결되는 파일이다.  
이 파일들은 jpo들을 repository에 선언한 SQL 메소드들을 활용하여 DB 항목의 물리적 값을 CRUD하는데 활용된다. 

```{.java}
package item.store;

import java.util.NoSuchElementException;
import java.util.Optional;

import org.springframework.stereotype.Repository;

import item.domain.entity.Item;
import item.domain.store.ItemStore;
import item.store.jpo.ItemJpo;
import item.store.repository.ItemRepository;

@Repository
public class ItemJpaStore implements ItemStore {

 private final ItemRepository itemRepository;

 public ItemJpaStore(ItemRepository itemRepository) {
  //
  this.itemRepository = itemRepository;
 }

 @Override
 public void create(Item item) {
  //
  itemRepository.save(new ItemJpo(item));
 }

 @Override
 public Item retrieve(String id) {
  //
  Optional<ItemJpo> itemJpo = itemRepository.findById(id);
  if (!itemJpo.isPresent()) {
   throw new NoSuchElementException("");
  }
  return itemJpo.get().toDomain();
 }

 @Override
 public void update(Item item) {
  //
  itemRepository.save(new ItemJpo(item));
 }

 @Override
 public void delete(String id) {
  //
  itemRepository.deleteById(id);
 }

 @Override
 public void delete(Item item) {
  //
  itemRepository.delete(new ItemJpo(item));
 }

}
```

### 4. Lifecycle  
Domain에 설계된 Sotre 패키지에 접근하기 위하여 다리 역할을 하는lifecycler를 활용하게 된다.  

```{.java}
package item.lifecycle;

import org.springframework.stereotype.Component;

import item.domain.lifecycle.StoreLifecycle;
import item.domain.store.ItemStore;

@Component
public class StoreLifecycler implements StoreLifecycle {
 private final ItemStore itemStore;
 
 public StoreLifecycler( ItemStore itemStore ){
  this.itemStore = itemStore;
 }
 
 @Override
 public ItemStore requestItemStore(){
  return this.itemStore;
 }
 
}
```
