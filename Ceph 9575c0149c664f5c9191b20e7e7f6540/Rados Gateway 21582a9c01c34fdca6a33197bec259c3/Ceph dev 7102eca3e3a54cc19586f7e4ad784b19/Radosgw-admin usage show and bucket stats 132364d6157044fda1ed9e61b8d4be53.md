# Radosgw-admin usage show and bucket stats

# 1. radosgw-admin usage show

Relevant class

- func process_request(…) in rgw_process.cc, it call the method rgw_log_op(…). This method handles a HTTP request from client, after finishing, it will decide if it should perform log operation by checking if the ***rgw_enable_usage_log*** is set to ***true***
- The dump process is performed 2 struct ***rgw_usage_log_entry*** and ***rgw_user_bucket***
- show function get the usage information with the function read_usage and read_all_usage depends on the option.
    
    ![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled.png)
    
- the purpose of the read_usage function are describes in the comment: “*Read usage information about this bucket from the backing store ”*
    
    ![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%201.png)
    
    The function read_usage is declared in the class **User** and **Bucket**.
    
    This function is an virtual function, and it is implemented in the class DBUser, which extends StoreUser, which extends User.
    
- Every time there is a request, it will be processed in the function **process_request** in the file **rgw_process.cc. I**n this function, after the request finishes then it call a function **rgw_log_op(…**).
    
    ![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%202.png)
    
    The **rgw_log_op(…)** function calculate the statistic for the operation (byte_send, byte_receive,…), then call the function **log(s, entry)** to dump the log to the driver.
    
    ![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%203.png)
    
    Here, we have 3 implementations of the log function
    
    ![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%204.png)
    
    ![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%205.png)
    
    **log_ops** function has several implementations, but only **RadosStore** driver 
    
    ![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%206.png)
    
    In the implementation of the **log_ops**, it get the information of the log_pool (which is stored in the zone configuration) and store the log into that pool with the method **append_async(…)**
    

## How the rgw usage show command retrieve log from the pool ?

The usage is actually stored in the omap of the **usage.13** object in the **log_pool**

![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%207.png)

The function that retrieves usage information is **read_usage(…).** This function is a virtual function of the **Bucket** and **User** class.

![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%208.png)

There are a lot of implementations (> 10) of the **read_usage(…)** function, but most of them are implemented as a procedure and have no function. For example,  

![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%209.png)

There are only 3 classes implementing that function.

![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%2010.png)

**RadosUser** extends **User** class, it call the function **getRados()** to return **RGWRados** object, which actually implement the logic of the **read_usage(…)** function

![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%2011.png)

![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%2012.png)

The cls_obj_usage_log_read(…) is the function that will get the usage from backend.

![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%2013.png)

this function get **usage_log_pool** from zone parameter. This function call a **cls call** to read the usage of the

![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%2014.png)

### **RGWSAL RGW Store Abstraction Layer**

![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%2015.png)

SAL stands for store abstract layer

In the SAL layer, Driver is the base abstraction of the SAL layer.

It use Singleton pattern, that allow only one object is created throughout the operation of ceph.

Driver contains User, Bucket and Object entities.

Beside User, in this layer, we have the class Bucket, Zone,… So, it can be regarded as the fundamental blocks of the rgw.

# Radosgw-admin bucket stats

- File rgw_admin.cc is where the command and logic of all radosgw-admin command is defined.

![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%2016.png)

The struct **RGWBucketAdminOpState** store information for bucket stat

![Untitled](Radosgw-admin%20usage%20show%20and%20bucket%20stats%20132364d6157044fda1ed9e61b8d4be53/Untitled%2017.png)