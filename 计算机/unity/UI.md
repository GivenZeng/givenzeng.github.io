# image
## 更换图片
```cs
using UnityEngine;
using System.Collections;
using UnityEngine.UI;
  
public class Test : MonoBehaviour
{
  
    [SerializeField]
    Image myImage;
  
    // Use this for initialization
    void Start()
    {
        myImage.sprite = Resources.Load("Image/pic", typeof(Sprite)) as Sprite;     // Image/pic 在 Assets/Resources/目录下
    }
}
```


## 图片点击
### 1
给Image增加一个Buttion component

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

public class StartPause : MonoBehaviour
{
    void Awake()
    {
        GetComponent<Button>().onClick.AddListener(click);
    }

    private void click()
    {
        Debug.Log("click");
    }
}
```