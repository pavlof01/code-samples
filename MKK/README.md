# Code samples of MKK project
### API Bridge RCT.m
```js
//
//  APIBridgeRCT.m
//  MKK
//
//  Created by Алексей Павлов on 20/12/2019.
//  Copyright © 2019 Facebook. All rights reserved.
//

#import <Foundation/Foundation.h>
#import "React/RCTBridgeModule.h"

@interface RCT_EXTERN_MODULE(APIBridgeIOS, NSObject)
RCT_EXTERN_METHOD(get:(NSString *)path
                  parameters:(NSDictionary *)parameters
                  resolver:(RCTPromiseResolveBlock)resolve
                  rejecter:(RCTPromiseRejectBlock)reject)

RCT_EXTERN_METHOD(post:(NSString *)path
                  parameters:(NSDictionary *)parameters
                  resolver:(RCTPromiseResolveBlock)resolve
                  rejecter:(RCTPromiseRejectBlock)reject)
@end

```

### API Bridge RCT.swift
```js
//
//  APIBridgeRCT.swift
//  MKK
//
//  Created by Алексей Павлов on 20/12/2019.
//  Copyright © 2019 Facebook. All rights reserved.
//

import Alamofire
import OHHTTPStubs

@objc(APIBridgeIOS)
class APIBridgeIOS: NSObject {

    private weak var apiClient = APIClient.shared
  
  @objc func get(_ path: String,
                 parameters: Parameters? = nil,
                 resolver resolve: @escaping RCTPromiseResolveBlock,
                 rejecter reject: @escaping RCTPromiseRejectBlock) {

    apiClient?.get(path, parameters: parameters, completion: { (result) in
      resolve(result.value)
    })
    
  }
  
  @objc func post(_ path: String,
                   parameters: Parameters? = nil,
                   resolver resolve: @escaping RCTPromiseResolveBlock,
                   rejecter reject: @escaping RCTPromiseRejectBlock) {
    
      apiClient?.post(path, parameters: parameters, completion: { (result) in
        resolve(result.value)
      })
      
    }
  
  
  @objc static func requiresMainQueueSetup() -> Bool {
    return true
  }
  
}

// MARK: - Constants
private extension APIBridgeIOS {
    
    enum Constants {
        
        static let path = "polls"
        static let stopPath = "polls/%i/stop"
        static let itemPath = "polls/%i"

        static let pollId = "id"
        static let archive = "archive"
        static let query = "filter"
        static let skip = "skip"
        static let take = "take"
    }
}
```
### API clien.m
```js
//
//  APIclien.m
//  MKK
//
//  Created by Алексей Павлов on 20/12/2019.
//  Copyright © 2019 Facebook. All rights reserved.
//

#import <Foundation/Foundation.h>
#import "React/RCTBridgeModule.h"

@interface RCT_EXTERN_MODULE(APIClient, NSObject)
RCT_EXTERN_METHOD(getApi:(NSString *)path
                  resolver:(RCTPromiseResolveBlock)resolve
                  rejecter:(RCTPromiseRejectBlock)reject)
// RCT_EXTERN_METHOD(get:(NSString)path parameters:(NSString)parameters)
@end

```
### API client.swift
```js
import Alamofire
import ObjectMapper
import AlamofireObjectMapper
import SwiftyJSON

// Backend request executor
class APIClient {

    static let shared = APIClient()
    
    private var session = SessionManager.default
  
    var taskDidReceiveChallengeWithCompletionHandler: ((URLSession, URLSessionTask, URLAuthenticationChallenge, @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void) -> Void)? {
        
        didSet {
            session.delegate.taskDidReceiveChallengeWithCompletion = taskDidReceiveChallengeWithCompletionHandler
        }
    }
  
  func get(_ path: String, parameters: Parameters? = nil, completion: @escaping (DataResponse<Any>) -> ()) -> Void {
      
        let urlToRequest = baseURLString + path
    
      session.request(urlToRequest,   parameters:parameters).responseJSON(completionHandler: { response in
        completion(response)
      })
  }
  
  func post(_ path: String, parameters: Parameters? = nil, completion: @escaping (DataResponse<Any>) -> ()) -> Void {
      
        let urlToRequest = baseURLString + path
    
      session.request(urlToRequest, method: .post, parameters:parameters, encoding: JSONEncoding.default).responseJSON(completionHandler: { response in
        completion(response)
      })
  }
}

// MARK: - Public interface
extension APIClient {
    
    func downloadFile(at path: String,
                      completion: @escaping (APIResult<Data>, String?, Double?) -> ()) -> DownloadRequest {
        
        let urlToRequest = baseURLString + path
        
        let destination = suggestedDownloadDestination()

        return session.download(urlToRequest, method: .get, to: destination)
        .validate(statusCode: [200])
        .responseData { response in
            
            var duration: Double?
            if let cache = response.response?.allHeaderFields["Cache-Control"] as? String {
                duration = cache.durationValue
            }
            
            var fileName: String?
            if let headers = response.response?.allHeaderFields {
                fileName = self.fileName(from: headers)
            }
            
            switch response.result {
                
            case .success(let value):
                completion(.success(value), fileName, duration)
            case .failure(let error):
                let apiError = APINetworkError(code: response.response?.statusCode ?? 0, originalError: error)
                completion(.failure(apiError), nil, nil)
            }
        }
    }

    func reset() {

        let configuration = URLSessionConfiguration.default
        configuration.urlCredentialStorage = nil
        session = SessionManager(configuration: configuration)
    }
}

// MARK: - Constants
private extension APIClient {
    
    var baseURLString: String {
        
        guard let urlKeyValue = Bundle.main.infoDictionary?["API Url"] as? String else {
            fatalError("No API url")
        }
        return urlKeyValue.replacingOccurrences(of: "\\", with: "")
    }
    
}

struct APINetworkError: LocalizedError {
 
    let code: Int
    let originalError: Error?

    var errorDescription: String? {

        return "\(code) \(originalError?.localizedDescription ?? "")"
    }
}

enum APIResult<ValueType> {
    case success(ValueType)
    case failure(APINetworkError)
}

```
### Authorization Provider IOS side
```js
//
//  AuthorizationProvider.m
//  MKK
//
//  Created by Алексей Павлов on 20/12/2019.
//  Copyright © 2019 Facebook. All rights reserved.
//

#import <Foundation/Foundation.h>
#import "React/RCTBridgeModule.h"

@interface RCT_EXTERN_MODULE(AuthorizationProvider, NSObject)
RCT_EXTERN_METHOD(login:(NSString)login
                  password:(NSString)password
                  resolver:(RCTPromiseResolveBlock)resolve
                  rejecter:(RCTPromiseRejectBlock)reject)
@end

```
###
```js
import Foundation
import KeychainAccess

@objc(AuthorizationProvider)
class AuthorizationProvider: NSObject {
    
    static let shared = AuthorizationProvider()
  
    var user: UserViewModel?
  
  private override init() {
    super.init()
        let hasStoredPinInfo = (try? keychain.contains(Constants.pinKey)) ?? false
        lockState = hasStoredPinInfo ? .locked : .noSet
    }
    
    fileprivate weak var apiClient = APIClient.shared
    fileprivate lazy var keychain = Keychain(service: Constants.serviceName)
    
    private var currentAttemptLogin: String?
    private var currentAttemptPassword: String?

    private var lockState: LockState = .noSet {

        didSet {
            postCurrentLockStateNotification()
        }
    }
  
  @objc func login(_ login: String,
                   password: String,
                   resolver resolve: @escaping RCTPromiseResolveBlock,
                   rejecter reject: @escaping RCTPromiseRejectBlock) {
      reset()

      currentAttemptLogin = login
      currentAttemptPassword = password

      apiClient?.taskDidReceiveChallengeWithCompletionHandler = taskDidReceiveChallengeWithCompletionHandler
    
    apiClient?.get(Constants.path, completion: { result in
    switch result.result {
      case .success:
        self.handleLoginSuccess(with: login, password: password)
        resolve(result.value)
      case .failure(_):
        reject("401", "Неправильный логин или пароль", result.error!)
      }
    })
    
  }
  
  @objc static func requiresMainQueueSetup() -> Bool {
    return true
  }
  
}

```
### API JS side
```js
import { NativeModules } from 'react-native'

import Api from './index'
import { IUser, INotificationEventType } from 'types/models'
import { isAndroid } from 'styles'

class User {
  static login(login: string, pass: string): Promise<IUser> {
    //TODO change bridging names
    return isAndroid ? NativeModules.Auth.login(login, pass) : NativeModules.AuthorizationProvider.login(login, pass)
  }

  static registerDeviceForNotifications(DeviceId: string, PushNotificationsToken: string) {
    return Api.post(`users/registerdevice`, { DeviceId, PushNotificationsToken })
  }

  static getMutedEvents() {
    return Api.get('users/muted_events')
  }

  static muteEvent(event: INotificationEventType) {
    return Api.post('users/mute_event', { EventType: event })
  }

  static unmuteEvent(event: INotificationEventType) {
    return Api.post('users/unmute_event', { EventType: event })
  }
}

export default User
```
### API Polls
```js
import Api from './index'

import { PollQuery } from 'types'
import { IVoteTypes } from 'types/models'

class Polls {
  /**
   * Получение списка заявок
   *
   * @param filter - Строка для поиска
   * @param archive - Признак архивности заявки
   * @param take - Максимальное количество возвращаемых элементов
   * @param skip - Количество пропускаемых элементов
   */

  static getPolls({ filter, archive, take, skip }: Partial<PollQuery>) {
    return Api.get('polls', { filter, archive, take, skip })
  }

  static getPollInfo(pollId: number) {
    return Api.get(`polls/${pollId}`)
  }

  /**
   * Получение голосов по заявке
   *
   * @param pollId - Уникальный идентификатор заявки
   */

  static getPollVotes(pollId: number) {
    return Api.get(`polls/${pollId}/votes`)
  }

  static vote(pollId: number, { VoteValue, Comment }: { VoteValue: IVoteTypes; Comment: string }) {
    return Api.post(`v2/polls/${pollId}/votes`, { VoteValue, Comment })
  }

  static getPollChatMessages(pollId: number) {
    return Api.get(`polls/${pollId}/chat`)
  }

  static sendMessage(pollId: number, text: String) {
    return Api.post(`polls/${pollId}/chat`, { Text: text })
  }

  static getParticiants(pollId: number) {
    return Api.get(`polls/${pollId}/addedparticipants`)
  }

  static getAllUsers() {
    return Api.get('users/list')
  }

  static addUser(pollId: number, UserId: number) {
    return Api.post(`polls/${pollId}/add_user`, { UserId })
  }

  static stopPoll(pollId: number) {
    return Api.post(`polls/${pollId}/stop`)
  }
}

export default Polls
```
### ROOT api file
```js
import { NativeModules } from 'react-native'

import { isAndroid } from 'styles'

const API_URL = 'https://mccvotetest.ocs.ru/api/'

class Api {
  /**
   * GET
   *
   * @param {string} url
   * @param {object} params
   * @return {AxiosPromise}
   */

  static get<T = any>(url: string, params: any = {}): Promise<T> {
    //TODO change bridging names
    return isAndroid ? NativeModules.Api.get(API_URL + url, params) : NativeModules.APIBridgeIOS.get(url, params)
  }

  /**
   * POST
   *
   * @param {string} url
   * @param {any} data
   * @param {AxiosRequestConfig} config
   * @return {AxiosPromise}
   */

  static post<T = any>(url: string, data: any = {}): Promise<T> {
    //TODO change bridging names
    return isAndroid
      ? NativeModules.Api.post(API_URL + url, JSON.stringify(data))
      : NativeModules.APIBridgeIOS.post(url, data)
  }
}

export default Api
```
### Auth sagas
```js
import { call, put, takeLatest, select } from 'redux-saga/effects'
import * as Keychain from 'react-native-keychain'

import AuthActions, { LOGIN, UPDATE_MUTED_EVENTS } from './actions'
import AuthApi from 'api/auth'
import { IStore } from 'types'
import { INotificationEventType } from 'types/models'
import { convertNativeObjectToJSObject } from 'utils'

function* loginFlow(action: ReturnType<typeof AuthActions.login>) {
  try {
    const { login, pass } = action.payload
    if (!login || !pass) return yield put(AuthActions.loginFail('Пожалуйста введите логин и пароль'))

    const user = yield call(AuthApi.login, login, pass)
    const mutedEvents = yield call(AuthApi.getMutedEvents)
    yield Keychain.setGenericPassword(login, pass)
    yield put(AuthActions.loginSuccess(convertNativeObjectToJSObject(user)))
    yield put(AuthActions.setMutedEvents(convertNativeObjectToJSObject(mutedEvents)))
  } catch (err) {
    console.warn(JSON.stringify(err.message, null, 2))
    yield put(AuthActions.loginFail(err.message))
  }
}

function* updateMutedEventsFlow(action: ReturnType<typeof AuthActions.updateMutedEvents>) {
  try {
    const getMutedEvents = ({ auth }: IStore) => auth.get('mutedEvents')
    const mutedEvents: INotificationEventType[] = yield select(getMutedEvents)
    const isMuted = mutedEvents.includes(action.payload)
    const mutedEventsData = yield call(isMuted ? AuthApi.unmuteEvent : AuthApi.muteEvent, action.payload)
    const mutedEventsDataParsed = convertNativeObjectToJSObject(mutedEventsData)
    if (mutedEventsDataParsed) yield put(AuthActions.setMutedEvents(mutedEventsDataParsed))
  } catch (error) {
    console.warn(error)
  }
}

export default function* watchAuths() {
  yield takeLatest(LOGIN, loginFlow)
  yield takeLatest(UPDATE_MUTED_EVENTS, updateMutedEventsFlow)
}
```
### Auth store
```js
import { fromJS, Record } from 'immutable'
import _ from 'lodash'

import {
  LOGIN,
  LOGIN_SUCCESS,
  LOGIN_FAIL,
  SET_USER,
  UPDATE_USER,
  SET_DEVICE_ID,
  SET_FCM_TOKEN,
  SET_MUTED_EVENTS,
  SET_PIN_STATUS,
  LOCKED,
  UNLOCKED,
  LOGOUT,
  UserTypes,
} from './actions'
import { IUser, INotificationEventType } from 'types/models'
import { IPinStatus } from 'types'

type IState = {
  loading: boolean
  isLogin: boolean
  isLocked: boolean
  pinStatus: IPinStatus
  user: IUser | null
  deviceId: string
  fcmToken: string
  mutedEvents: INotificationEventType[]
  error: string | null
}

const initialState: Record<IState> = fromJS({
  loading: false,
  isLogin: false,
  isLocked: false,
  pinStatus: 'choose',
  user: null,
  deviceId: null,
  fcmToken: null,
  mutedEvents: [],
  error: null,
})

export const auth = (state = initialState, action: UserTypes): typeof initialState => {
  const user = state.get('user')
  switch (action.type) {
    case LOGIN:
      return state.set('loading', true)
    case LOGIN_SUCCESS:
      return state
        .set('user', action.payload)
        .set('isLogin', true)
        .set('loading', false)
        .set('error', null)
    case LOGIN_FAIL:
      return state.set('error', action.payload).set('loading', false)
    case SET_USER:
      return state.set('user', action.payload).set('isLogin', true)
    case UPDATE_USER:
      return state.set('user', { ...user, ...action.payload })
    case SET_DEVICE_ID:
      return state.set('deviceId', action.payload)
    case SET_FCM_TOKEN:
      return state.set('fcmToken', action.payload)
    case SET_MUTED_EVENTS:
      return state.set('mutedEvents', action.payload)
    case SET_PIN_STATUS:
      return state.set('pinStatus', action.payload)
    case LOCKED:
      return state.set('isLocked', true)
    case UNLOCKED:
      return state.set('isLocked', false)
    case LOGOUT:
      return fromJS(initialState)
    default:
      return state
  }
}

export default auth
```
### API bridge android side
```js
package ru.ocs.creditcommitteevoteandroid_dev;

import android.util.Log;
import java.io.IOException;
import org.json.JSONObject;

import com.facebook.react.bridge.Promise;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.bridge.ReactContextBaseJavaModule;
import com.facebook.react.bridge.ReactMethod;
import com.facebook.react.bridge.ReadableMap;

public class Api extends ReactContextBaseJavaModule {
    private static ReactApplicationContext reactContext;
    private static final String CLASS_TAG = "Api";
    static String LOGIN = "";
    static String PASSWORD = "";

    Api(ReactApplicationContext context) {
        super(context);
        reactContext = context;
    }

    public static String setLogin(String login) { return LOGIN= login; }

    public static String setPassword(String password) {
        return PASSWORD = password;
    }

    @Override
    public String getName() {
        return "Api";
    }

    @ReactMethod
    public void get(String url, ReadableMap data, Promise promise) throws IOException {
        try {
//            Log.i(CLASS_TAG, LOGIN);
            String response =  HttpConn.getString(url, LOGIN, PASSWORD, data);
            promise.resolve(response);
        }
        catch(Exception e) {
            promise.reject(e);
        }

    }

    @ReactMethod
    public void post(String url, String data, Promise promise) throws IOException {

        try {
            String response =  HttpConn.post(url, LOGIN, PASSWORD, data);
            promise.resolve(response);
        }
        catch(Exception e) {
            promise.reject(e);
        }

    }
}
```
### API REACT PACKAGE
```js
package ru.ocs.creditcommitteevoteandroid_dev;

import com.facebook.react.ReactPackage;
import com.facebook.react.bridge.NativeModule;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.uimanager.ViewManager;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class ApiPackage implements ReactPackage {

    @Override
    public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
        return Collections.emptyList();
    }

    @Override
    public List<NativeModule> createNativeModules(
            ReactApplicationContext reactContext) {
        List<NativeModule> modules = new ArrayList<>();

        modules.add(new Api(reactContext));

        return modules;
    }
}
```
### Auth android side
```js
package ru.ocs.creditcommitteevoteandroid_dev;

import android.accounts.AccountManager;
import android.accounts.Account;

import com.facebook.react.bridge.Promise;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.bridge.ReactContextBaseJavaModule;
import com.facebook.react.bridge.ReactMethod;

import java.io.IOException;
import java.net.URL;

import okhttp3.OkHttpClient;
import okhttp3.Response;
import okhttp3.Request;

public class Authenticate extends ReactContextBaseJavaModule {
    private static ReactApplicationContext reactContext;
    private static final String endPoint = "AUTH_URL";

    Authenticate(ReactApplicationContext context) {
        super(context);
        reactContext = context;
    }

    @Override
    public String getName() {
        return "Auth";
    }

    @ReactMethod
    public void login(String login, String password, Promise promise) {
        try {
            OkHttpClient client = HttpConn.getClient(login, password).build();

            Request.Builder requestBuilder = new Request.Builder();
            URL url = new URL(endPoint);
            requestBuilder.url(url);

            Response response = client.newCall(requestBuilder.build()).execute();

            if (!response.isSuccessful())
                throw new IOException(response.message());
            if (response.body() != null) {
                Api.setLogin(login);
                Api.setPassword(password);
                promise.resolve(response.body().string());
            }
        }

        catch (Exception e) {
            promise.reject(e.getMessage(), e.getMessage());
            e.printStackTrace();
        }

    }

}
```
### Auth react package
```js
package ru.ocs.creditcommitteevoteandroid_dev;

import com.facebook.react.ReactPackage;
import com.facebook.react.bridge.NativeModule;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.uimanager.ViewManager;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class AuthenticatePackage implements ReactPackage {

    @Override
    public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
        return Collections.emptyList();
    }

    @Override
    public List<NativeModule> createNativeModules(
            ReactApplicationContext reactContext) {
        List<NativeModule> modules = new ArrayList<>();

        modules.add(new Authenticate(reactContext));

        return modules;
    }
}
```
