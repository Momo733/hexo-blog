---
title: golang服务自动化测试与生成接口文档
date: 2019-06-09 20:53:37
tags:
 - DevOps
 - apitest
 - generate doc
 - gin
 - yuque
 - golang
 - test
---
写完服务API就到了自己测试接口，并且写接口文档的时候。在继承自动化编译部署阶段的时候，我在想如果能够自动化的生成文档，那简直是太棒了，这肯定是会节省大量的时间。自动化测试我们的golang服务，其实官方已经有``net/http/httptest/``包来生成一个server和request，帮助我们测试自己的请求，并且使用recorder来获取到请求和响应的数据。如果能够在编写集成测试接口的时候，使用``go test``，把记录保存下来生成文档，那么是一箭双雕的事情，可以节省大量的时间。

## 使用apitest来模拟请求并且记录数据

[apitest](https://github.com/steinfletcher/apitest)是一个封装了``net/http/httptest/``功能的测试库，方便我们生成序列图，封装了请求参数使用JSON，body等方法直接请求，还有自定义Format接口可以生成一切想要的数据格式。

apitest官方支持的框架类型如下：

| Example                                                                                              | Comment                                                                                                    |
| ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| [gin](https://github.com/steinfletcher/apitest/tree/master/examples/gin)                             | popular martini-like web framework                                                                         |
| [gorilla](https://github.com/steinfletcher/apitest/tree/master/examples/gorilla)                     | the gorilla web toolkit                                                                                    |
| [iris](https://github.com/steinfletcher/apitest/tree/master/examples/iris)                           | iris web framework                                                                                         |
| [echo](https://github.com/steinfletcher/apitest/tree/master/examples/echo)                           | High performance, extensible, minimalist Go web framework                                                  |
| [mocks](https://github.com/steinfletcher/apitest/tree/master/examples/mocks)                         | example mocking out external http calls                                                                    |
| [sequence diagrams](https://github.com/steinfletcher/apitest/tree/master/examples/sequence-diagrams) | generate sequence diagrams from tests. See the [demo](http://demo-html.apitest.dev/)  |

我们可以自己添加自己的server，并且设置一些需要运行参数的中间件，也可以隔离一些例如权限检查的中间件，添加自己所写的Handler，方便我们自己进行测试。

下面是使用apitest实现的一个集成测试用例，``docs.RecorderCaptor{}``实现了Fromat接口，``docs.GinServerInit()``是初始化一个gin的server，自定义中添加handler使用``server.AddGetHandler``或者``server.AddPostHandler``，其余的都是配置的一些参数，方便生成文档。
```
func TestRobotRulesJhListHandler(t *testing.T) {
	reporter := &docs.RecorderCaptor{}
	server := docs.GinServerInit()
	//************************修改区域******************
	var relativePath = "/v1/robot/rulejh/list"
	var param = map[string]string{"page": "1", "rows": "1"}
	//var postparam = map[string]interface{}{"id":1}
	reporter.RequestParam = param
	reporter.Description = "查询" + config.FunctionNameMapList["api.RobotRulesJhListHandler"]
	reporter.Title = "后台管理文档"
	reporter.SubTitle = "## " + config.FunctionNameMapList["api.RobotRulesJhListHandler"]
	server.AddGetHandler(relativePath, api.RobotRulesJhListHandler)
	apitest.New("RobotRulesJhListHandler").
		//************************修改区域******************
		Report(reporter).
		Handler(server.Engine).
		Get(relativePath).
		QueryParams(param).
		Expect(t).
		Status(http.StatusOK).
		End()
}
```

## 自定义Format接口

在apitest中使用repoter就可以把所需要的数据记录到参数中，在Report方法中实现的是一个ReportFormatter接口，官方实现是使用``SequenceDiagramFormatter``来生成一个序列图，如果想要生成Markdown文档也是非常方便，我们直接可以自定义Format接口。

```
func (a *APITest) Report(reporter ReportFormatter) *APITest {
	a.reporter = reporter
	return a
}

ReportFormatter interface {
    Format(*Recorder)
}
```

下面是我自己实现的Format接口，RecorderCaptor是可以用于传递一些测试用例中的自己设置的参数，在Format中可以看到获取的Recorder有四种类型，通常我们只需要HttpRequest和HttpResponse两个类型，MessageRequest和MessageResponse是用于mock时候的数据记录。

```
type RecorderCaptor struct {
	capturedRecorder apitest.Recorder
	RequestParam     interface{}
	param            []Param
	Description      string
	Title            string
	SubTitle         string
}

func (r *RecorderCaptor) Format(recorder *apitest.Recorder) {
	var doc = &Document{}
	r.CheckParam()
	doc.Request.Param = r.param
	doc.Description = r.Description
	doc.SubTitle = r.SubTitle
	doc.Title=r.Title
	for _, event := range recorder.Events {
		switch v := event.(type) {
		case apitest.HttpRequest:
			httpReq := v.Value
			doc.ParseParam(httpReq)
		case apitest.HttpResponse:
			httpRes := v.Value
			doc.ParseResponse(httpRes)
		case apitest.MessageRequest:
			continue
		case apitest.MessageResponse:
			continue
		default:
			panic("received unknown event type")
		}
	}
	//推送文档
	doc.Generate()
}
```

## 文档部署

文档既可以生成在本地，也可以直接推送到远程的一些文档服务器，比如语雀，因为是使用gitlab来推送代码更新接口的时候生成，所以可以使用CD命令来部署，也是也之前部署项目是相同的，只是推送到服务的是一些文档或者序列图，Markdown格式文档有hexo和hugo来解析生成，序列图直接浏览即可。

由于公司购买了语雀的空间，所以使用语雀的api接口，就可以直接把文档推送到远程，但是语雀的更新文档是非常的鸡肋，比如更新文档中的某一部分，需要先获取文档全部的类容，如果文档内容少点还好，多的话就会更新就需要去匹配到指定的内容，然后把修改后的内容推送给语雀，不能很方便的部分修改。

所以建议还是使用公司内部开始一个文档服务器，这样既可以防止文档泄露，也能方便的直接部署文档直接替换，每次更新不必在意生成文档等太久。