###Android Appium自动化测试指导手册
####环境配置
mac下环境配置：http://www.cnblogs.com/oscarxie/p/3894559.html
windows下环境配置类似，主要是JDK、AndroidSDK配置，略
####新建项目
本次主要介绍用java来写appium自动化case，测试项目搭建方法如下：
1.使用eclipse/intellij新建java project
2.拷贝三个基础jar包至libs目录下，jar包具体如下：java-client-3.1.0.jar、junit-			4.7.jar、selenium-server-standalone-2.46.0.jar，各包均拿最新版本即可
3.新建testcase即可，使用junit测试框架，具体的一个完整的测试case如下：

	public class TakeScreenShotDemo {
    	private AppiumDriver driver;

    	@Before
    	// 初始化测试环境
    	public void setUp() throws Exception {
        	driver = (AppiumDriver)SingletonDriver.getDriverInstance();
    	}

    	@After
    	// 后置操作
    	public void tearDown() throws Exception {
        	driver.quit();
    	}

    	@Test
    	public void demoTest() throws InterruptedException {
        	NovaTestUtils.openApp(driver);

        	//截屏
        	NovaTestUtils.takeScreenShot(driver, "/Users/horizon/Downloads/");

        	//查找元素
        	WebElement el = driver.findElementByName("上海");

        	//断言
        	assertThat(el.getText(), equalTo("上海"));

        	// 等待控件出现
        	WebElement e1 = new WebDriverWait(driver, 15).until(ExpectedConditions.presenceOfElementLocated(By.name("上海")));

        	// 等待控件出现，并获取控件
        	WebElement e = (new WebDriverWait(driver, 10)).until(new ExpectedCondition<WebElement>() {
            	@Override
            	public WebElement apply(WebDriver d) {
                return d.findElement(By.id("上海"));
            	}
       		});

    	}
	}


####appium driver配置
	public static WebDriver getDriverInstance() throws MalformedURLException {
        if (singletonDriver == null) {
            // set up appium
            File classpathRoot = new File(System.getProperty("user.dir"));
            // 存放app目录：apps
            File appDir = new File(classpathRoot, "apps");
            File app = new File(appDir, "Nova-common-debug.apk");
            DesiredCapabilities capabilities = new DesiredCapabilities();
            capabilities.setCapability("deviceName", "192.168.57.101");
            // capabilities.setCapability(CapabilityType.BROWSER_NAME, "");
            // 模拟器的安卓版本是4.4
            capabilities.setCapability(CapabilityType.VERSION, "4.4");
            capabilities.setCapability("platformName", "Android");
            //让appium支持输入中文
            capabilities.setCapability("unicodeKeyboard", "True");
            capabilities.setCapability("resetKeyboard", "True");

            capabilities.setCapability("app", app.getAbsolutePath());
            capabilities.setCapability("app-package", "com.dianping.v1");
            // capabilities.setCapability("app-activity",
            // ".activity.MainActivity");
            singletonDriver = new AppiumDriver(new URL("http://127.0.0.1:4723/wd/hub"),
                    capabilities) {
                @Override
                public WebElement scrollTo(String s) {
                    return null;
                }

                @Override
                public WebElement scrollToExact(String s) {
                    return null;
                }
            };
        }

        return singletonDriver;
    }


####常用方法
1.通过id、name等查找控件(包括listview中的元素、dialog中的元素统一可使用此方式查找)

	WebElement url = driver.findElement(By.id("com.dianping.v1:id/text_url"));
	url.sendKeys(UrlScheme);

	WebElement mock = driver.findElement(By.name("mock"));
	mock.click();

2.点击控件

	WebElement mock = driver.findElement(By.name("mock"));
	mock.click();

3.等待控件出现

	WebElement e1 = new WebDriverWait(driver, 15).until(ExpectedConditions.presenceOfElementLocated(By.name("上海")));

4.等待控件出现，并获取控件

	WebElement e = (new WebDriverWait(driver, 10)).until(new ExpectedCondition<WebElement>() {
            @Override
            public WebElement apply(WebDriver d) {
                return d.findElement(By.id("上海"));
            }
        });

5.输入中文
首先设置Capabilities

	DesiredCapabilities capabilities = new DesiredCapabilities();
	//让appium支持输入中文
	capabilities.setCapability("unicodeKeyboard", "True");
	capabilities.setCapability("resetKeyboard", "True");

然后在自动化case中调用

	//输入中文
	WebElement inputTextToSearch1 = 	driver.findElement(By.id("com.dianping.v1:id/search_edit"));
	inputTextToSearch1.sendKeys("韩膳宫料理 中山公园店");

6.滑动屏幕（滑动到最上方、最下方）

滑动到最上方

	/**
 	 * This Method for swipe down
 	 *
 	 * @param driver
 	 * @param during
 	 */
	public static void swipeToDown(AppiumDriver driver, int during) {
    	int width = driver.manage().window().getSize().width;
    	int height = driver.manage().window().getSize().height;
    	driver.swipe(width / 2, height * 3 / 4, width / 2, height / 4, during);
    	// wait for page loading
	}

滑动到最下方

	/**
 	 * This Method for swipe up
 	 *
 	 * @param driver
 	 * @param during
  	 */
	public static void swipeToUp(AppiumDriver driver, int during) {
    	int width = driver.manage().window().getSize().width;
    	int height = driver.manage().window().getSize().height;
    	driver.swipe(width / 2, height / 4, width / 2, height * 3 / 4, during);
    	// wait for page loading
	}

7.截屏

	/**
 	 * 截屏
 	 * @param driver
 	 * @param desFilePath 全路径，如/Users/horizon/Downloads/
 	 */
	public static void takeScreenShot(AppiumDriver driver, String desFilePath) {
		File screenShotFile = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
		try {
			FileUtils.copyFile(screenShotFile,
            new File(desFilePath + getCurrentDateTime() + ".png"));
		} catch (IOException e) {
			e.printStackTrace();
		}
	}

	public static String getCurrentDateTime() {
		SimpleDateFormat df = new SimpleDateFormat("yyyyMMdd_HHmmss");// 设置日期格式
		return df.format(new Date());
	}

8.结果校验

	//断言
	assertThat(el.getText(), equalTo("上海"));

