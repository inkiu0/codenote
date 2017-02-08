# 测试环境
1. MacMini + Unity5.3.6
2. trunk/Develop
3. 服务器[18]前端开发
4. 测试流程从登录到在南门随意走动  
# Profiler剖析
从图中可以看出只是取**RoleRender_ChangeColor.running**就占了ms，%。  
原代码如下
```  
public bool running
{
    get
    {
        return current != to;
    }
}
```  
# 优化方案
```  
public bool running
{
    get
    {
        return current.r != to.r;
    }
}
```  
# 性能测试
```
//测试代码如下
bool m_running = false;
float time = 0;
float totalTime = 0;
int count = 0;
System.Diagnostics.Stopwatch stopwatch = new System.Diagnostics.Stopwatch();
public bool running
{
	get
	{
		stopwatch.Start ();
        //m_running = current != to;
        //m_running = current.r != to.r || current.g != to.g || current.b != to.b || current.a != to.a;
		m_running = timeElapsed < duration;
		stopwatch.Stop ();
		totalTime += stopwatch.ElapsedMilliseconds;
		count += 1;
		if (count == 1000)
		{
			Debug.Log ("totalTime = " + totalTime + " timeFromStart =  " + Time.realtimeSinceStartup + " per = " + (totalTime / Time.realtimeSinceStartup * 1000));
		}
		return m_running;
	}
}
```  
分别测试了调用1000次/5000次/10000次消耗的性能，并和优化后的代码作比较，得出下表。