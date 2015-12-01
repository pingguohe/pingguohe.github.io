---

layout: post
title: Android单元测试初探
author: 赵林

--- 

Android下有很多单元测试的框架，这里简单介绍一下我最近使用的两个，android SDK自带的单元测试框架和Robolectric。

###AndroidTestCase
AndroidTestCase使用JUnit框架进行单元测试，首先需要在gradle中进入依赖

    testCompile 'junit:junit:4.12'
    androidTestCompile 'com.android.support.test:runner:0.4'
这里一定要引入`com.android.support.test:runner`。然后在gradle中配置AndroidJUnitRunner

	defaultConfig {
			minSdkVersion 14
			targetSdkVersion Integer.parseInt(System.properties['compileSdkVersion'])
			versionCode 1
			versionName "1.0"
			testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
		}
如果运行的时候出现runner找不到的情况，检查一下这里有没有配置。然后新建一个测试类`TMConfigProcessUtilsTest`，代码如下

	public class TMConfigProcessUtilsTest extends AndroidTestCase {
	
		@Before
		public void setUp() throws Exception {
	
		}
	
		@After
		public void tearDown() throws Exception {
	
		}
	
		@Test
		public void testIsMainProcess() throws Exception {
			assertEquals(true, TMConfigProcessUtils.isMainProcess(getContext()));
		}
	}
测试代码继承自AndroidTestCase，可以在setUp()方法中初始化一些变量。如果直接使用JUnit是无法获取context的，也就无法对android的代码进行测试，继承AndroidTestCase之后可以直接使用getContext()方法获取context。然后需要将Test Artifact切换成Android Instrumentation test。直接运行测试代码，结果如下。
![ut1](/images/2015/12/image_1.png)
会直接显示测试代码运行的结果，也可以导出到html中进行查看。

###Robolectric
Robolectric 是一款Android单元测试框架，它可以直接运行在JVM之上，不需要真机或者模拟器。首先需要在gradle引入Robolectric的依赖。  

	testCompile "org.robolectric:robolectric:3.0"  

新建一个测试类`TMConfigStringUtilsTest`，代码如下

	@RunWith(RobolectricGradleTestRunner.class)
	@Config(constants = BuildConfig.class, sdk=21)
	public class TMConfigStringUtilsTest {

		@Before
		public void setUp() throws Exception {
	
		}
	
		@After
		public void tearDown() throws Exception {
	
		}
	
		@Test
		public void testIsEmpty() throws Exception {
			Assert.assertEquals(true, TMConfigStringUtils.isEmpty(""));
		}
	}  
利用注解，指定TestRunner为RobolectricGradleTestRunner，之后在@Condig指定constants和SDK版本。如果这里不指定sdk版本，并且当前sdk版本高于21时会抛出java.lang.UnsupportedOperationException异常，这里很诡异，完整的异常是
	
	java.lang.UnsupportedOperationException: Robolectric does not support API level 22.

Robolectric不支持API Level 22，所以这里必须加上sdk=21或其它支持的版本，指定sdk的版本。然后需要将Test Artifact切换成unit test。如果需要获取application可以直接调用`RuntimeEnvironment.application`。

配置完成后在命令行中运行
	
	gradle test

就会自动运行所有测试代码，如果不希望被某一个错误用例的断言中断运行，可以在后面加上`--continue`。运行完成后，会在build/reports/tests/release/index.html中生成运行结果。
![ut2](/images/2015/12/image_2.png)
这里可以直接查看每个类运行的情况。

###对比
对android单元测试接触的时间比较短，简单做下对比。  
+ AndroidTestCase需要实际的android环境，需要真机或者模拟器，一个字慢！Robolectric不需要android环境，运行起来很快。
+ Robolectric虽然可以用`RuntimeEnvironment.application`获取application，但是从实际的运行结果来看这里应用并没有启动，所以就无法获取当前的进程名，在AndroidTestCase中的那个例子在Robolectric中是跑不过，在测试一些与运行时有关的方法可能会有问题。而继承自AndroidTestCase的测试代码需要在实际的系统环境中运行，没有这个问题。

###可能遇到的问题
+ 找不到AndroidJUnitRunner，除了在gradle中配置，也需要在android studio中检查一下是不是已经配置了
![ut3](/images/2015/12/image_3.png)
+ 在android studio中代码不可用。注意检查Test Artifact选择是否正确。android studio新建一个工程默认的文件结构是  
![ut4](/images/2015/12/image_4.png)  
androidTest中默认是与android相关的单元测试，即Android Instrumentation test，test默认是JUnit的测试代码，使用Robolectric需要切换到unit test。