#  Axios

导出的是一个default的axios对象



这个对象是通过createInstance实现的

``` javascript
function createInstance(defaultConfig) {
    const context = new Axios(defaultConfig); //创建一个Axios实例
    const instance = bind(Axios.prototype.request, context);
    
    utils.extend(instance, Axios.prototype, context, {allOwnKeys: true});
    
    utils.extend(instance, context, null, {allOwnKeys: true});
    
    instance.create = function create(instanceConfig) {
        return createInstance(mergeConfig(defaultConfig, instanceConfig));
    }
    
    return instance;
}

const axios = createInstance(defaults);

class Axios {
    constructor(instanceConfig) {
        this.defaults = instanceConfig;
        this.interceptors = {
            request: new InterceptorManager(),
            response: new InterceptorManager()
        }
    }
    
    request() {}
    getUri() {}
}

class InterceptorManager {
    constructor() {
        this.handlers = [];
    }
    
    use(fulfilled, rejected, options) {}
    
    eject(id) {}
    
    clear() {}
    
    forEach(fn) {}
}

export default axios;
```



手动实现一个Axios

``` javascript
class Axios {
    constructor() {
        this.interceptors = {
            request: new InterceptorsManage(),
            response: new InterceptorsManage()
        }
    }
    
    request(config) {
        let chain = [this.sendAjax.bind(this), undefined];
        this.interceptors.request.handlers.forEach((interceptor) => {
            chain.unshift(interceptor.fullfilled, interceptor.rejected)
        })
        this.interceptors.response.handlers.forEach((interceptor) => {
            chain.push(interceptor.fullfilled, interceptor.rejected);
        })
        let promise = Promise.resolve(config);
        while(chain.length > 0) {
            promise = promise.then(chain.shift(), chain.shift())
        }
        return promise;
    }
    sendAjax(config) {
        return new Promise((resolve) => {
            const xhr = new XMLHttpRequest();
            const {url='', method='get', data = {}} = config;
            xhr.open(method, url, true);
            xhr.onload = function() {
                resolve(xhr.responseText);
            }
            xhr.send(data);
        })
    }
}

function CreateAxios() {
    const axios = new Axios();
    const req = axios.request.bind(axios);
    //把添加的方法绑定到axios上
    utils.extend(req, Axios.prototype, axios);
    //去把interceptors绑定到request上
    utils.extend(req, axios);
    return req;
}

const methodsArr = ['get', 'delete', 'head', 'options', 'put', 'post'];
methodsArr.forEach((item)=>{
    Axios.prototype[item] = function() {
        if(['head', 'get', 'delete', 'options'].includes(item)) {
            return this.request({
                method: item, 
                url: arguments[0],
                ...arguments[1] || {}
            })
        }else {
            return this.request({
                method: item,
                url: arguments[0],
                data: arguments[1] || {},
                ...arguments[2] || {}
            })
        }
    }
})
//但是此时的方法是在Axios上，导出的是Axios.request，写个方法把这些方法绑定到request上
//把b的方法绑定到a上
const utils = {
    extend(a, b, context) {
        for(let key in b) {
            if(b.hasOwnProperty(key)) {
                if(typeof b[key] === 'function') a[key] = b[key].bind(context);
                else a[key] = b[key];
            }
        }
    }
}
```

接下来去实现拦截器

``` javascript
class InterceptorsManage {
    constructor() {
        this.handlers = [];
    }
    
    use(fullfilled, rejected) {
        this.handlers.push({
            fullfilled,
            rejected
        })
    }
}
```

