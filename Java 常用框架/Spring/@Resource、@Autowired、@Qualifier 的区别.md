# @Resource、@Autowired、@Qualifier 的区别

@Resource 和 @Autowired 都是用于类型注入，前者是通过按 name 匹配，后者是通过按 type 匹配的

@Resource 的装配顺序：
> @Resource 有两个属性，分别是 name 和 type
> 1. 如果同时指定了 name 和 type，spring 会从容器中找到唯一匹配的 bean 进行装配，找不到则抛出异常
> 2. 如果指定了 name 值，则从容器查找 name 匹配的 bean 执行装配，找不到则抛出异常
> 3. 如果指定 type 属性值，则从容器中查找类型匹配的唯一 bean 进行装配，找不到或找到多个都会抛出异常
> 4. 如果都不指定，则会自动按照 byName 进行装配，如果没有匹配，则回退一个原始类型进行装配，如果匹配则自动装配	

@Autowired 的装配顺序
> 按照类型匹配装入，默认情况要求必须存在，除非设置 required 为 false。如果存在多个会抛出异常，可以通过 @Qualifier 指定名称
