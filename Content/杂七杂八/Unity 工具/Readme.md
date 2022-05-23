视频链接 https://www.youtube.com/watch?v=pZ45O2hg_30
 
![image](https://user-images.githubusercontent.com/29577919/169755513-dbdb70e2-f220-4182-809d-0dfffb943d4a.png)
- 在onenable和disable的时候自动加入static组
- execute always可以在不play的时候也运行

```
//barrel
    private void OnEnable() => ExplosiveBarrelManager.allTheBarrels.Add(this); //use this to automate, very useful!
    private void OnDisable() => ExplosiveBarrelManager.allTheBarrels.Remove(this); //expression body members

    private void OnDrawGizmosSelected() => Gizmos.DrawWireSphere(transform.position, radius);
```
```
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
```
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
