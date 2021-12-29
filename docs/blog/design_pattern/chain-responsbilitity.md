# 责任链模式 <!-- {docsify-ignore-all} -->


## ChainHandler

```java
public interface ChainHandler extends Ordered {

    /**
     * 执行
     */
    void execute(Chain chain);

    /**
     * 是否跳过
     * @return true / false
     */
    boolean skip();
}
```

## ChainHandler1和ChainHandler2

```java
public class ChainHandler1 implements ChainHandler {

    @Override
    public void execute(Chain chain) {
        System.out.println("This is filter 1");
        chain.execute();
    }

    @Override
    public boolean skip() {
        return false;
    }

    @Override
    public int getOrder() {
        return 1;
    }
}

public class ChainHandler2 implements ChainHandler {

    @Override
    public void execute(Chain chain) {
        System.out.println("This is filter 2");
        chain.execute();
    }

    @Override
    public boolean skip() {
        return true;
    }

    @Override
    public int getOrder() {
        return 2;
    }
}

```

## 责任链入口

```java
public interface Chain {

    /**
     * 执行
     */
    void execute();
}
```

```java
public class FilterChain implements Chain {

    public static void main(String[] args) {
        List<ChainHandler> chainHandlers = new ArrayList<>();
        ChainHandler chainHandler1 = new ChainHandler1();
        ChainHandler chainHandler2 = new ChainHandler2();
        chainHandlers.add(chainHandler1);
        chainHandlers.add(chainHandler2);
        Chain chain = new FilterChain(chainHandlers);
        chain.execute();
    }

    private int index;

    private final List<ChainHandler> chainHandlers;

    public FilterChain(List<ChainHandler> chainHandlers) {
        this.chainHandlers = chainHandlers;
    }

    @Override
    public void execute() {
        if (index < chainHandlers.size()) {
            ChainHandler chainHandler = chainHandlers.get(index++);
            boolean skip = chainHandler.skip();
            if (skip) {
                this.execute();
                return;
            }
            chainHandler.execute(this);
        }
    }
}
```