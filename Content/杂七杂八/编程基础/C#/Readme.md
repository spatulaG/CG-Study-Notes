## 究竟什么是序列化？
```
    public float thing;//serialized, visible, public 
    private float thing0;//not serialized, hidden, private
    [SerializeField] float thing1;//private, visible, serialized
    [HideInInspector] private float thing2;//serialized, hidden, public
    // try to keep everything inside a serializable custom type serialized 
```

- Stable, long-term data, opposite of runtime data
- Subproperties should be serialized as well
- No inheritance, polymorphism...
