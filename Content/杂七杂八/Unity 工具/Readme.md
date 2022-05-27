视频链接 https://www.youtube.com/watch?v=pZ45O2hg_30
 
![image](https://user-images.githubusercontent.com/29577919/169755513-dbdb70e2-f220-4182-809d-0dfffb943d4a.png)
- 在onenable和disable的时候自动加入static组
- execute always可以在不play的时候也运行

```c#
//barrel
    private void OnEnable() => ExplosiveBarrelManager.allTheBarrels.Add(this); //use this to automate, very useful!
    private void OnDisable() => ExplosiveBarrelManager.allTheBarrels.Remove(this); //expression body members

    private void OnDrawGizmosSelected() => Gizmos.DrawWireSphere(transform.position, radius);
```
```c#
//Manager
void OnDrawGizmosSelected()
    {
        foreach (var barrel in allTheBarrels)
        {
            #if UNITY_EDITOR
            Vector3 managerPos = transform.position;
            Vector3 barrelPos = barrel.transform.position;
            float halfPos = (managerPos.y - barrelPos.y)*.5f;
            Vector3 halfOffset = halfPos * Vector3.up;

            Handles.DrawBezier(transform.position, barrel.transform.position,managerPos - halfOffset, barrelPos + halfOffset, Color.blue,Texture2D.whiteTexture,1f);
            #endif
        }
    }
```
生成动态材质
```c#
private void Awake()
    {
        Shader shader = Shader.Find("Default");
        Material mat = new Material(shader) { hideFlags = HideFlags.HideAndDontSave };// will destroy upon exit
        
        // duplicate Material = extra draw call, can't batch, leak
        GetComponent<MeshRenderer>().material.color = Color.red;
        // will modify asset
        GetComponent<MeshRenderer>().sharedMaterial.color = Color.red;
    }
```

## 添加颜色
![image](https://user-images.githubusercontent.com/29577919/169761986-a17732f6-7c88-4e9a-92ee-8e513430554a.png)

```
 MaterialPropertyBlock mpb;
    static readonly int shPropColor = Shader.PropertyToID("_Color");
    public MaterialPropertyBlock Mpb
    {
        get {
            if (mpb == null) 
                mpb = new MaterialPropertyBlock();
            return mpb;
        }
    }

    void ApplyColor()
    {
        MeshRenderer rnd = GetComponent<MeshRenderer>();
        Mpb.SetColor(shPropColor, color);
        rnd.SetPropertyBlock(Mpb);
    }

    private void OnValidate()// everytime it detects changes
    {
        ApplyColor();
    }
```
## 生成自定义asset类型
![image](https://user-images.githubusercontent.com/29577919/169767541-890bac77-ea91-44a8-a0b8-f0be27e2db22.png)

```c#
public class BarrelType : ScriptableObject
{
    [Range(1f, 8f)]
    public float radius = 1;

    public float damage = 10f;
    public Color color = Color.red;
}
```

## Property 的使用
```c#
//外部脚本只能访问而不能修改其值
    public bool IsGameOver { get; private set; } // 设为 auto property
//或
    public bool IsGameOver
    {
        get
        {
            return isGameOver; // 读取对应数值
        }

        set
        {
            isGameOver = value; // 赋值操作，这里 value 的数据类型需要和目标一致，这里为 bool
        }

    }
```
