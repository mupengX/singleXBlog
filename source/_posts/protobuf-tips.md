title: 利用Protocol buffers FieldMask和Wrapper Message设计API

date: 2021-10-23 16:30:42

categories: Go

tags: Go

------



## 前言

在做API设计的时候经常遇到两个问题：

1、更新接口参数是否可以只传要修改的字段。

2、protobuf3 Go generated code中没有HasField()方法之后，如何判断某个字段是否被赋值。

## 实践

### FieldMask

对于前面提到的第一个问题可以利用FieldMask来解决，看下FieldMask官方介绍：

> `FieldMask` represents a set of symbolic field paths. Field masks are used to specify a subset of fields that should be returned by a get operation (a *projection*), or modified by an update operation.

FieldMask是一组字段的路径，它可以用来指定返回哪些字段或更新哪些字段。例如：我们有个一个更新用户信息的接口，在更新接口中我们可以通过FieldMask指定更新哪些字段，这样就不必为每个字段的更新设计一个接口。

Protobuf 定义如下：

```go
import "google/protobuf/field_mask.proto";
import "google/protobuf/wrappers.proto";
message User {
  int64 id = 1;
  string name = 2;
  string gender = 4;
  string birthday = 5;
  google.protobuf.StringValue email = 6;
  google.protobuf.DoubleValue money = 7;
  google.protobuf.Int32Value size = 8;
}

message UpdateUserRequest {
  User user = 1;
  google.protobuf.FieldMask field_mask = 2;
}

message UserReply {
  User user = 1;
}

service UserService {
  rpc UpdateUser(UpdateUserRequest) returns (UserReply);
  rpc FindUser(FindUserRequest) returns (UserReply);
}
```

Client端在请求时指定要更新的字段，未被指定的字段不会被更新。代码如下：

```go
func updateUser(client pb.UserServiceClient) {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	user := &pb.User{
		Id:       1001,
		Name:     "Jerry",
		Gender:   "female",
		Birthday: "1991-01-01",
		Money:    &wrappers.DoubleValue{Value: 123243.43},
	}
	req := &pb.UpdateUserRequest{
		User: user,
		FieldMask: &field_mask.FieldMask{
			Paths: []string{"user.birthday", "user.money"},
		},
	}
	reply, err := client.UpdateUser(ctx, req)
	log.Printf("update user reply: %v, err: %v\n", reply, err)
}
```

Server 端根据请求中的FieldMask来更新响应字段。代码如下：

```go
func (s *server) UpdateUser(ctx context.Context, req *pb.UpdateUserRequest) (*pb.UserReply, error) {
	// assume is found by req.GetUser().GetId() from db
	dst := &pb.User{
		Id:       1001,
		Name:     "mary",
		Gender:   "female",
		Birthday: "1996-12-12",
		Email:    &wrappers.StringValue{Value: "xxx@xx.com"},
		Money:    &wrappers.DoubleValue{Value: 12.34},
		Size:     &wrappers.Int32Value{Value: 12},
	}
	log.Printf("user find from db: %v\n", dst)
	fmutils.Filter(req, req.GetFieldMask().GetPaths())
	src := req.GetUser()
	log.Printf("user to be merged: %v\n", src)
	proto.Merge(dst, src)
	log.Printf("user to update: %v\n", dst)
	// omit save to db
	return &pb.UserReply{
		User: dst,
	}, nil
}
```

Client端请求日志：

```shell
2021/10/30 15:41:21 update user reply: user:{id:1001  name:"mary"  gender:"female"  birthday:"1991-01-01"  email:{value:"xxx@xx.com"}  money:{value:123243.43}  size:{value:12}}, err: <nil>
```

Server端日志：

```shell
2021/10/30 15:41:21 user find from db: id:1001  name:"mary"  gender:"female"  birthday:"1996-12-12"  email:{value:"xxx@xx.com"}  money:{value:12.34}  size:{value:12}
2021/10/30 15:41:21 user to be merged: birthday:"1991-01-01"  money:{value:123243.43}
2021/10/30 15:41:21 user to update: id:1001  name:"mary"  gender:"female"  birthday:"1991-01-01"  email:{value:"xxx@xx.com"}  money:{value:123243.43}  size:{value:12}
```

可以看到通过FieldMask只更新了用户的birthday和money两个字段，其他字段未被更新。这样做的好处是所有字段的更新可以统一到一个接口中，不需要根据不同的字段定义不同的接口，例如：UpdateUserBirthDay、UpdateUserMoney等。

也可以使用FieldMask来指定查找接口里的返回字段：

Client端代码：

```go
func main() {
	conn, err := grpc.Dial("127.0.0.1:8888", grpc.WithInsecure(), grpc.WithBlock())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()

	client := pb.NewUserServiceClient(conn)

	updateUser(client)
	// find user without field mask
	findUser(client, nil)
	// find user with field mask
	fieldMask := &field_mask.FieldMask{
		Paths: []string{"id", "name", "gender", "email"},
	}
	findUser(client, fieldMask)
}

func findUser(client pb.UserServiceClient, fieldMask *field_mask.FieldMask) {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	req := &pb.FindUserRequest{
		Id:        1001,
		FieldMask: fieldMask,
	}
	reply, err := client.FindUser(ctx, req)
	log.Printf("find user reply: %v, fieldMask: %v, err: %v\n", reply, fieldMask, err)
}
```

Server端代码：

```go
func (s *server) FindUser(ctx context.Context, req *pb.FindUserRequest) (*pb.UserReply, error) {
	user := &pb.User{
		Id:       1001,
		Name:     "mary",
		Gender:   "female",
		Birthday: "1996-12-12",
		Email:    &wrappers.StringValue{Value: "xxx@xx.com"},
		Money:    &wrappers.DoubleValue{Value: 12.34},
		Size:     &wrappers.Int32Value{Value: 12},
	}
	log.Printf("field mask path: %v\n", req.GetFieldMask().GetPaths())
	// filter user protobuf message by field mask
	fmutils.Filter(user, req.GetFieldMask().GetPaths())
	return &pb.UserReply{
		User: user,
	}, nil
}
```

Client端请求日志：

```she
2021/10/30 16:00:20 find user reply: user:{id:1001  name:"mary"  gender:"female"  birthday:"1996-12-12"  email:{value:"xxx@xx.com"}  money:{value:12.34}  size:{value:12}}, fieldMask: <nil>, err: <nil>
2021/10/30 16:00:20 find user reply: user:{id:1001  name:"mary"  gender:"female"  email:{value:"xxx@xx.com"}}, fieldMask: paths:"id"  paths:"name"  paths:"gender"  paths:"email", err: <nil>
```

Server端日志：

```shel
2021/10/30 16:00:20 field mask path: []
2021/10/30 16:00:20 field mask path: [id name gender email]
```

可以看到第一次FieldMask未赋值则User所有字段都返回，第二次请求则返回了指定字段。

### Wrapper Message

针对第二个问题，除了FieldMask外注意到上面的代码中User Message中email、money等部分字段使用了 wrapper message类型而非基础类型来定义，这样做的好处是可以清晰的区分出该字段是未被赋值还是被赋为零值。例如：请求中money这个字段通过moneyl==0无法判断moeny这字段是未赋值还是赋为空。

例如：检查更新用户的接口中money字段必须赋值，否则报错

```go
func (s *server) validateUpdateUserRequest (ctx context.Context, req *pb.UpdateUserRequest) bool {
	if req.GetUser().GetMoney() == nil {
		return false
	}
	return true
}
```

当然也可以在protobuf中使用protoc-gen-validate等工具来限定money字段不为空

## 参考

[Protobuf FieldMask](https://developers.google.cn/protocol-buffers/docs/reference/google.protobuf?hl=zh-cn#fieldmask)

[Protobuf Wrappers](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/wrappers.proto)

[Protobuf fieldmask util](https://github.com/mennanov/fieldmask-utils)

[Netflix API Design](https://netflixtechblog.com/practical-api-design-at-netflix-part-1-using-protobuf-fieldmask-35cfdc606518)

