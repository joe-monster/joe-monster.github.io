	
#### 我个人觉得这个没找到数据的错误，不应该是个错误，或者说出现这个错误的方法或函数内应该是有能力处理这个错误的，不应该往上抛。所以，我一般是忽略这个错误，把其他错误往上抛，用这次学习到的 pkg/errors 包，把gorm返回的错误用warp加一下调用栈信息，额外加一点有用的信息再往上抛，上层就不用再warp了，只需要加点信息就可以了，最后在api层记录一次日志即可。

以下代码之前初学时写的，凑合拿出来做个例子，垃圾代码，轻喷。。。



#### model函数

```

import (
	...
	pkgerrors "github.com/pkg/errors"
)

func TemplateInfo(id string) (interface{}, error) {

	templateNodes := []work_order_process.TemplateNode{}
	if err := pgsql.DbWorkOrderProcess.Where("template_id = ? and delete_time = ?", id, "").Find(&templateNodes).Error; err != nil {
		if err != sql.ErrNoRows {
			return nil, pkgerrors.Wrap(err, fmt.Sprintf("模版节点数据查询异常 template_id:%s", id))
		}
	}

	templateCc := []work_order_process.TemplateCc{}
	if err := pgsql.DbWorkOrderProcess.Where("template_id = ? and delete_time = ?", id, "").Find(&templateCc).Error; err != nil {
		if err != sql.ErrNoRows {
			return nil, pkgerrors.Wrap(err, fmt.Sprintf("模版抄送数据查询异常 template_id:%s", id))
		}
	}

	type rtnType struct {
		Nodes []nodeType `json:"nodes"`
		Cc    []ccType   `json:"cc"`
	}
	var rtn rtnType

	for _, v := range templateNodes {
		...
		rtn.Nodes = append(rtn.Nodes, obj)
	}

	for _, v := range templateCc {
		...
		rtn.Cc = append(rtn.Cc, obj)
	}

	return rtn, nil
}
```

#### service函数

```
import (
	...
	pkgerrors "github.com/pkg/errors"
)

func TemplateInfo(id string) (interface{}, error) {
	data, err := model.TemplateInfo(id)
	if err != nil {
		return data, pkgerrors.WithMessage(err, "service异常")
	}
	return data, nil
}
```


#### api函数

```
//获取单条详情
func Info(c *qing.Context) {
	id := c.Param("id")

	data, err := service.TemplateInfo(id)
	if err != nil {
		log.Printf("%+v 参数[id:%v] \n", err, id)
		c.JSON(http.StatusOK, mze.Error(e.ERROR, err))
		return
	}

	c.JSON(http.StatusOK, mze.Success(data))
}
```