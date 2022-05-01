1.UserMapper.xml: wrong mapper namespace(DebugContest.DebugContest.Mapper.UserMapper)

2.UserMapper.java:lack @Mapper on UserMapper

3.UserServiceImpl.java:lack @Service on UserServiceImpl

4.UserController.java:lack @Autowired on UserService

5.UserController.java:wrong requestMapping(/use->/user)

6.application.yml:server port(8081->8080)

7.debugcontest.sql: email(int->varchar)

8.UserController.java:register:wrong mapping(/login->/register)

9.application.yml:lack address(I use localhost.)

10.UserController.java:login:wrong Mapping(PostMapping->GetMapping)

11.UserController.java:register:wrong param anno and url(@PathVariable->@RequestParam)

12.application.yml:wrong mapper-location(*Mapper.xml->XML/\*.xml)

13.UserServiceImpl.java:register: lack inserting

14.UserServiceImpl.java: wrong returning value(success->Success)

15.UserMapper.xml: wrong inserting(password,email->email,password)

16.UserServiceImpl.java:login: wrong if case(==0->!=0)

17.UserController.java: wrong anno(@Controller->@RestController)

18.UserMapper.xml:selectByEmail: wrong selecting(email->count(*))

