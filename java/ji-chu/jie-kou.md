# 接口

接口的实现可以不依托实现类，直接implement method 可以实现接口

```
test test = new test() {
            @Override
            public void helloworld() {
                
            }
};
```

要是接口只有一个方法，可以通过lamda表达式表达



```
test test = () -> {    System.out.println("helloworld");    };
```
