# frontend_guides

## Caching
### Simple in memory cache
```
export function memoizeMemoryAsync(func, cacheNamespace = '') {
  const inMemoryCache = {};
  return async (...myArgs) => {
    const cacheKey = `inMemoryCache.${cacheNamespace}.${JSON.stringify(myArgs)}`;
    if (inMemoryCache[cacheKey] === undefined) {
      // get from async
      try {
        inMemoryCache[cacheKey] = func.apply(null, myArgs);
      } catch (e) {}
    }

    return await inMemoryCache[cacheKey];
  };
}
```



### Memoize with LocalStorage
```
import StorageUtils from './StorageUtils';

const CACHE_KEY_PREFIX = 'myAppCache.';

export function memoizeLocalStorageAsync(func, cacheNamespace = '') {
  return async (...myArgs) => {
    const cacheKey = `${CACHE_KEY_PREFIX}${cacheNamespace}.${JSON.stringify(myArgs)}`;
    if (StorageUtils.getJson(cacheKey) === undefined) {
      // get from async
      try {
        const resp = await func.apply(null, myArgs);
        StorageUtils.setJson(cacheKey, resp);
      } catch (e) {}
    }

    return StorageUtils.getJson(cacheKey);
  };
}
```

### Busting Cache
```
const CACHE_KEY_TIMESTAMP = 'myAppCacheLastTimestamp';

function _burstCache() {
  const nucleusCacheLastTimestamp = parseInt(SessionStorageUtils.get(CACHE_KEY_TIMESTAMP, '0'));
  if (Date.now() - nucleusCacheLastTimestamp >= toClearCacheTimerMs) {
    // clear cache
    SessionStorageUtils.clear();
    SessionStorageUtils.set(CACHE_KEY_TIMESTAMP, Date.now());
  }

  setTimeout(_burstCache, toClearCacheTimerMs); // rerun after the above timer
}

_burstCache(); // run once for each page refresh...
```


## StorageUtils
### LocalStorageUtils
```
export const LocalStorageUtils = {
  clear() {
    window.localStorage.clear();
  },
  setJson(key, val) {
    window.localStorage[key] = JSON.stringify(val);
  },
  getJson(key, defaultVal) {
    try {
      return JSON.parse(window.localStorage[key]);
    } catch (e) {}
    return defaultVal;
  },
  set(key, val) {
    window.localStorage[key] = val;
  },
  get(key, defaultVal = '') {
    return window.localStorage[key] || defaultVal;
  },
  getAsBoolean(key, defaultVal = '') {
    return (window.localStorage[key] || defaultVal).toLowerCase() === 'true';
  },
};
```

### SessionStorageUtils
```
export const SessionStorageUtils = {
  clear() {
    window.sessionStorage.clear();
  },
  setJson(key, val) {
    window.sessionStorage[key] = JSON.stringify(val);
  },
  getJson(key, defaultVal) {
    try {
      return JSON.parse(window.sessionStorage[key]);
    } catch (e) {}
    return defaultVal;
  },
  set(key, val) {
    window.sessionStorage[key] = val;
  },
  get(key, defaultVal = '') {
    return window.sessionStorage[key] || defaultVal;
  },
  getAsBoolean(key, defaultVal = '') {
    return (window.sessionStorage[key] || defaultVal).toLowerCase() === 'true';
  },
};

```

## Deferred Promise
```
export function getDeferredPromise() {
  return (() => {
    let resolve;
    let reject;

    let p = new Promise((res, rej) => {
      resolve = res;
      reject = rej;
    });

    return {
      promise: p,
      reject,
      resolve,
    };
  })();
}
```


## Debounce Async
```
export const debounceAsync = (fnToUse, debounceTime = 300) => {
  let timerForDebounce = null;
  let lastDeferred = null;

  return (...rest) => {
    lastDeferred && lastDeferred.reject();
    timerForDebounce && clearTimeout(timerForDebounce);

    lastDeferred = getDeferredPromise();

    timerForDebounce = setTimeout(() => {
      fnToUse(...rest).then(lastDeferred.resolve, lastDeferred.resolve);
    }, debounceTime);

    return lastDeferred.promise;
  };
};
```


## Upload multipart form
### JS Code to upload
```
export const doTestUpload = (fileInputDom) =>
  new Promise((resolve) => {
    var xhr = new XMLHttpRequest();
    xhr.open('POST', `/api/server/attachment_test`);
    xhr.withCredentials = true;
    xhr.addEventListener('readystatechange', function() {
      this.readyState === 4 && resolve(this.responseText);
    });

    // set up the request to send
    const formData = new FormData();
    formData.append('file', fileInputDom.files[0]); // only allow 1 file at a time
    xhr.send(formData);
  });
```

### HTML Code
```
<input type="file" id="testFileUploadInput" />
```

### JS Caller
```
const fileInputDom = document.querySelector('#testFileUploadInput');
const newFileContent = await doTestUpload(fileInputDom);
```

### Server Code with Java SpringBoot
```
 @PostMapping(value = "/api/server/attachment_test", produces = APPLICATION_JSON_VALUE)
  public static ResponseEntity<InputStreamResource> attachmentTest(
      @RequestParam("file") final MultipartFile file) throws IOException {
    if (file == null || file.getContentType() == null) {
      throw new BadRequestException("File not specified");
    }
    MediaType contentType = APPLICATION_OCTET_STREAM;
    final String fileContentType = file.getContentType();
    if (fileContentType != null) {
      contentType = MediaType.parseMediaType(fileContentType);
    }
    return ResponseEntity.ok()
        .contentType(contentType)
        .contentLength(file.getSize())
        .body(new InputStreamResource(file.getInputStream()));
  }
```

## logger utils
```
export const log = (...rest) => {
  console.log('LOG: ', ...rest);
};

export const error = (...rest) => {
  console.error('ERROR: ', ...rest);
};

export const warn = (...rest) => {
  console.log('WARN: ', ...rest);
};

export const debug = (...rest) => {
  console.debug('DEBUG: ', ...rest);
};
```


## REST API and Fetch
```

/**
 *
 * @param {*} url url
 * @param {*} params request params
 * @param {*} postProcessingHandler function to be called after API returns (error / success), with params (res, url, params)
 * this can return a promise postProcessingHandler(res, url, params)
 */
export const doAbortableFetch = (url, params, postProcessingHandler) => {
  const controller = new AbortController();
  const signal = controller.signal;

  params = Object.assign(
    {
      headers: {
        accept: 'application/json, text/plain, */*',
        'Content-Type': 'application/json',
      },
      mode: 'cors',
      signal,
    },
    params,
  );

  const promiseFetch = fetch(url, params).then((res) => {
    const newHeaders = {};
    for (let [headerKey, headerValue] of res.headers) {
      newHeaders[headerKey] = headerValue;
    }
    res.parsedHeaders = newHeaders;

    if (res.ok) {
      return res;
    }

    // dispatch logging out for forbidden status...
    if (postProcessingHandler === undefined) {
      switch (parseInt(res.status)) {
        case 404: // not found
          throw new Error(`Record not found (${url})`);
        case 401: // (unauthorized) Log out , no retry
          throw new Error(
            `Unauthorized. Logging out. statusCode=${res.status} statusText=${res.statusText}`,
          );
        case 403: // (forbidden) Handle No permission to do something
          throw new Error(`You do not have sufficient permission to access this Record (${url})`);
        case 500: // (server errors)
        case 502: // (gateway nginx errors)
        default:
          // other errors...
          throw new Error(
            `Server Error. Redirecting to ErrorPage. statusCode=${res.status} statusText=${res.statusText}`,
          );
      }
    } else {
      // else call the postProcessingHandler
      return postProcessingHandler(res, url, params);
    }
  });

  return [
    promiseFetch,
    () => {
      // abort
      controller.abort();
    },
  ];
};

/**
 *
 * @param {*} url url
 * @param {*} params request params
 * @param {*} postProcessingHandler function to be called after API returns (error / success), with params (res, url, params)
 * this can return a promise postProcessingHandler(res, url, params)
 */
export const doFetch = (url, params, postProcessingHandler) => {
  let [promiseFetchTicket] = doAbortableFetch(url, params, postProcessingHandler);
  return promiseFetchTicket;
};
```
