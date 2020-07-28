```
        Human human1 =new Human();
        human1.setIdCardNum("0123456789");
        Human human2 =new Human();
        human2.setIdCardNum("0123456789");

        System.out.println(human1.hashCode());
        System.out.println(human2.hashCode());
        System.out.println(human1.equals(human2));
```

```
  @Override
  public int hashCode() {
    return Objects.hash(name,idCardNum,sex);
  }
```

虽然是两个不同对象，当往HashMap中插入人的时候，即使是两个不同对象，但是身份证一致则可以认为是同一个人，可以覆盖插入。

![](/assets/hashcode结果.png)

