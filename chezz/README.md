# Code samples of Chezz project

### Orders screen
```js
import React, { useState, useEffect } from 'react'
import { RefreshControl, FlatList } from 'react-native'
import { useSelector } from 'react-redux'
import { DotIndicator } from 'react-native-indicators'
import _ from 'lodash'

import { SAV, BoxFill, Text, H4, Image, theme, ScrollView } from 'styles'
import { IStore, IOrder, Snaphot } from 'types'
import { database, chefsRef } from 'firebase'

import Header from 'components/Header'
import OrderItem from 'components/Order'

const MyOrders = () => {
  const user = useSelector(({ auth }: IStore) => auth.get('user'))
  const [orders, setOrders] = useState<{ activeOrders: IOrder[]; archivedOrders: IOrder[] }>({
    activeOrders: [],
    archivedOrders: [],
  })
  const [loading, setLoading] = useState(true)
  const [refreshing, setRefreshing] = useState(false)

  useEffect(() => {
    getOrders()
  }, [])

  const getOrders = async () => {
    setRefreshing(true)
    const orders: IOrder[] = []
    const activeOrders: IOrder[] = []
    const archivedOrders: IOrder[] = []
    const ordersData = await database
      .ref('orders')
      .orderByChild('customerId')
      .equalTo(user!.uid)
      .once('value')
    ordersData.forEach((order: Snaphot<IOrder>) => orders.push({ key: order.key, ...order.val() }))
    for (const i of orders) {
      const chefData = await chefsRef.child(i.chefId).once('value')
      if (i.status === 'In progress') activeOrders.push({ ...i, chef: chefData.val() })
      if (i.status === 'Done') archivedOrders.push({ ...i, chef: chefData.val() })
    }
    setOrders({ activeOrders, archivedOrders })
    setLoading(false)
    setRefreshing(false)
  }

  if (loading)
    return (
      <SAV>
        <Header title="My Orders" />
        <DotIndicator color={theme.colors.main} count={3} size={10} />
      </SAV>
    )

  if (!orders.activeOrders.length && !orders.archivedOrders.length)
    return (
      <SAV>
        <Header title="My Orders" />
        <BoxFill mt="30%" alignItems="center">
          <Image source={require('../../assets/images/no_orders.png')} />
          <H4 mt={30} mb={14}>
            No Orders
          </H4>
          <Text textAlign="center" px="15%">
            Your current and past orders will be displayed here
          </Text>
        </BoxFill>
      </SAV>
    )

  return (
    <SAV>
      <Header title="My Orders" />
      <ScrollView refreshControl={<RefreshControl onRefresh={getOrders} refreshing={refreshing} />} px={14} py={20}>
        {orders.activeOrders.length ? (
          <BoxFill>
            <Text mb={8}>Active</Text>
            <FlatList
              contentContainerStyle={{ paddingBottom: 30 }}
              data={orders.activeOrders}
              ItemSeparatorComponent={() => <BoxFill my={15} />}
              renderItem={({ item }) => <OrderItem order={item} />}
            />
          </BoxFill>
        ) : null}
        {orders.archivedOrders.length ? (
          <BoxFill>
            <Text mb={8}>Completed</Text>
            <FlatList
              contentContainerStyle={{ paddingBottom: 30 }}
              data={orders.archivedOrders}
              ItemSeparatorComponent={() => <BoxFill my={15} />}
              renderItem={({ item }) => <OrderItem order={item} />}
            />
          </BoxFill>
        ) : null}
      </ScrollView>
    </SAV>
  )
}

export default MyOrders
```
### Profile screen
```js
import React from 'react'
import { Alert } from 'react-native'
import { useSelector, useDispatch } from 'react-redux'
import { NavigationInjectedProps } from 'react-navigation'

import { SAV, BoxFullFill, Box, BoxRow, Text, Image, H3, Touchable, BoxFill } from 'styles'
import { IStore } from 'types'
import { auth } from 'firebase'
import AuthActions from 'store/auth/actions'

import Button from 'components/Button'

const MENU = [
  { title: 'My Orders', routeName: 'MyOrders', icon: require('../../assets/icons/my_orders.png') },
  { title: 'Support', routeName: 'Support', icon: require('../../assets/icons/support.png') },
  { title: 'Become a chef', routeName: 'BecomeAchef', icon: require('../../assets/icons/become_a_chef.png') },
]

const Profile: React.FC<NavigationInjectedProps> = ({ navigation }) => {
  const dispatch = useDispatch()
  const user = useSelector(({ auth }: IStore) => auth.get('user'))
  const isLogin = useSelector(({ auth }: IStore) => auth.get('isLogin'))

  const logOut = () => {
    Alert.alert(
      'Log out',
      'Are you sure to log out?',
      [
        {
          text: 'Log out',
          onPress: () => {
            auth.signOut()
            dispatch(AuthActions.logout())
          },
          style: 'destructive',
        },
        { text: 'Cancel', onPress: () => {}, style: 'default' },
      ],
      { cancelable: true }
    )
  }

  const UnauthorizedScreen = () => {
    return (
      <BoxFullFill justifyContent="center" alignItems="center">
        <Image mb={35} source={require('../../assets/images/notLogin.png')} />
        <Text>Log in to start ordering food</Text>
        <Button title="Log in" onPress={() => navigation.navigate('SignIn')} w={190} mt={20} />
      </BoxFullFill>
    )
  }

  const ProfileScreen = () => {
    if (!user) return null
    return (
      <BoxFullFill px={14} pt={40}>
        <BoxRow alignItems="center">
          <Box bg="whiteAlt" width={62} height={62} borderRadius={31}>
            <Image source={require('../../assets/images/avatar.png')} />
          </Box>
          <Box ml={14}>
            <H3 fontWeight="bold">{user.displayName}</H3>
            <Touchable onPress={() => logOut()}>
              <Text color="main">Log out</Text>
            </Touchable>
          </Box>
        </BoxRow>
        <BoxFill mt={40}>
          {MENU.map(item => (
            <Touchable key={item.title} onPress={() => navigation.navigate(item.routeName)}>
              <BoxRow py={24} alignItems="center" borderBottomWidth={1}>
                <Image mr={15} source={item.icon} />
                <Text>{item.title}</Text>
              </BoxRow>
            </Touchable>
          ))}
        </BoxFill>
      </BoxFullFill>
    )
  }

  return <SAV>{isLogin ? ProfileScreen() : UnauthorizedScreen()}</SAV>
}

export default Profile
```
### SignIn screen
```js
import React, { useState } from 'react'
import firebase from '@react-native-firebase/app'
import { NavigationInjectedProps } from 'react-navigation'

import { SAV, Text, BoxFill, Image, H1, H7, TextInput, KeyboardAvoidingView, Touchable } from 'styles'

import Button from 'components/Button'

const SignIn: React.FC<NavigationInjectedProps> = ({ navigation }) => {
  const [phone, setPhone] = useState<any>()
  const [error, setError] = useState<any>()
  const [isLoading, setIsLoading] = useState(false)
  const logIn = () => {
    setIsLoading(true)
    firebase
      .auth()
      .signInWithPhoneNumber('+' + phone)
      .then(confirmResult => navigation.navigate('VerificationNumber', { confirmResult, phone }))
      .catch(error => setError(error.code))
      .finally(() => setIsLoading(false))
  }
  return (
    <SAV>
      <KeyboardAvoidingView justifyContent="space-between">
        <BoxFill px={14}>
          <Touchable my={25} onPress={() => navigation.goBack()}>
            <Image source={require('../../assets/icons/close.png')} />
          </Touchable>
          <H1 fontWeight="bold" textAlign="left">
            Sign in
          </H1>
          <H7 textAlign="left" mb={30} mt={6}>
            Whether you’re creating an account or signing back in, let’s start with your number
          </H7>
          <Text mb={10} color="error">
            {error}
          </Text>
          <TextInput
            keyboardType="phone-pad"
            value={phone}
            onChangeText={setPhone}
            autoFocus
            fontSize={35}
            color="gray"
            fontWeight="bold"
            placeholder="000 000 000"
          />
        </BoxFill>
        <BoxFill px={14} mb={10}>
          <BoxFill mb={10} alignItems="center">
            <Text>By signing in I accept Chez </Text>
            <Touchable onPress={() => navigation.navigate('TermsAndConditions')}>
              <Text style={{ textDecorationLine: 'underline' }} color="main">
                Terms and Conditions
              </Text>
            </Touchable>
          </BoxFill>
          <Button title="Sign in" onPress={logIn} disable={!phone} loading={isLoading} />
        </BoxFill>
      </KeyboardAvoidingView>
    </SAV>
  )
}

export default SignIn
```
### Create redux action helpers
```js
type FunctionType = (...args: any[]) => any
interface IActionCreatorsMapObject {
  [actionCreator: string]: FunctionType
}

export type ActionsUnion<A extends IActionCreatorsMapObject> = ReturnType<A[keyof A]>

export interface IAction<T extends string> {
  type: T
}

export interface IActionWithPayload<T extends string, P> extends IAction<T> {
  payload: P
}

export function createAction<T extends string>(type: T): IAction<T>
export function createAction<T extends string, P>(type: T, payload: P): IActionWithPayload<T, P>
export function createAction<T extends string, P>(type: T, payload?: P) {
  return payload === undefined ? { type } : { type, payload }
}
```
### Meal actions
```js
import { createAction, ActionsUnion } from '../helpers'
import { IMeal, Snaphot } from 'types'
import { database, chefsRef } from 'firebase'

export const SET_LOADING = 'SET_LOADING'
export const SET_READY_TO_EAT = 'SET_READY_TO_EAT'
export const SET_GROCERIES = 'SET_GROCERIES'
export const LOAD_MORE_READY_TO_EAT = 'LOAD_MORE_READY_TO_EAT'
export const LOAD_MORE_GROCERIES = 'LOAD_MORE_GROCERIES'

const NUM_INIT_ITEMS = 3

async function getReadyToEat(limit: number = NUM_INIT_ITEMS) {
  let readyToEat: IMeal[] = []
  await database
    .ref('meal')
    .orderByChild('status')
    .equalTo('active')
    // .limitToLast(limit)
    .once('value')
    .then(snapshot =>
      snapshot.forEach(
        (e: Snaphot<IMeal>) => e.val().type === 'Ready to eat' && readyToEat.push({ key: e.key, ...e.val() })
      )
    )
  for (const iterator of readyToEat) {
    const isAvailable = await chefsRef
      .child(iterator.chefid)
      .child('isAvailable')
      .once('value')
    if (!isAvailable.val()) readyToEat = readyToEat.filter(meal => iterator.chefid !== meal.chefid)
  }
  return readyToEat
}

async function getGroceries(limit: number = NUM_INIT_ITEMS) {
  let groceries: IMeal[] = []
  await database
    .ref('meal')
    .orderByChild('status')
    .equalTo('active')
    // .limitToFirst(limit)
    .once('value')
    .then(snapshot =>
      snapshot.forEach((e: Snaphot<IMeal>) => e.val().type === 'Grocery' && groceries.push({ key: e.key, ...e.val() }))
    )
  for (const iterator of groceries) {
    const isAvailable = await chefsRef
      .child(iterator.chefid)
      .child('isAvailable')
      .once('value')
    if (!isAvailable.val()) groceries = groceries.filter(meal => iterator.chefid !== meal.chefid)
  }
  return groceries
}

export async function refreshGroceries() {
  const groceries: IMeal[] = await getGroceries()
  return (dispatch: any) => {
    dispatch(MealActions.setGroceries(groceries))
  }
}
export async function refreshReadyToEat() {
  const readyToEat: IMeal[] = await getReadyToEat()
  return (dispatch: any) => {
    dispatch(MealActions.setReadyToEat(readyToEat))
  }
}

export async function loadMoreReadyToEat(count: number) {
  const readyToEat: IMeal[] = await getReadyToEat(count)
  return (dispatch: any) => {
    dispatch(MealActions.loadMoreReadyToEat(readyToEat))
  }
}

export async function loadMeal() {
  const readyToEat: IMeal[] = await getReadyToEat()
  const groceries: IMeal[] = await getGroceries()
  return (dispatch: any) => {
    dispatch(MealActions.setReadyToEat(readyToEat))
    dispatch(MealActions.setGroceries(groceries))
  }
}

const MealActions = {
  setLoading: (loading: boolean) => createAction(SET_LOADING, loading),
  setReadyToEat: (meal: IMeal[]) => createAction(SET_READY_TO_EAT, meal),
  setGroceries: (meal: IMeal[]) => createAction(SET_GROCERIES, meal),
  loadMoreReadyToEat: (meal: IMeal[]) => createAction(LOAD_MORE_READY_TO_EAT, meal),
  loadMoreGroceries: (meal: IMeal[]) => createAction(LOAD_MORE_GROCERIES, meal),
}

export type MealTypes = ActionsUnion<typeof MealActions>

export default MealActions
```
### Meal store
```js
import { fromJS, Record } from 'immutable'

import { SET_LOADING, SET_READY_TO_EAT, SET_GROCERIES, LOAD_MORE_READY_TO_EAT, MealTypes } from './actions'
import { IMeal } from 'types'

type IInitialState = {
  loading: boolean
  readyToEat: IMeal[]
  groceries: IMeal[]
}

const initialState: Record<IInitialState> = fromJS({
  loading: false,
  readyToEat: [],
  groceries: [],
})

export const Meal = (state = initialState, action: MealTypes): typeof initialState => {
  const readyToEat = state.get('readyToEat')
  const groceries = state.get('groceries')
  switch (action.type) {
    case SET_LOADING:
      return state.set('loading', action.payload)
    case SET_READY_TO_EAT:
      return state.set('readyToEat', action.payload)
    case LOAD_MORE_READY_TO_EAT:
      return state.set('readyToEat', readyToEat.concat(action.payload))
    case SET_GROCERIES:
      return state.set('groceries', action.payload)
    default:
      return state
  }
}

export default Meal
```
### Styles
```js
import React from 'react'
import { Dimensions, Platform, StatusBar } from 'react-native'
import RNFastImage, { FastImageProperties } from 'react-native-fast-image'
import styled from 'styled-components/native'
import { space, layout, color, typography, flexbox, borders, position } from 'styled-system'
import {
  ColorProps,
  SpaceProps,
  LayoutProps,
  FlexboxProps,
  TypographyProps,
  BordersProps,
  PositionProps,
} from 'styled-system'
import { isIphoneX } from '../utils'
import theme from './theme'

export { theme }

export const isAndroid = Platform.OS === 'android'

const { width, height } = Dimensions.get('window')
const [shortDimension, longDimension] = width < height ? [width, height] : [height, width]

const guidelineBaseWidth = 350
const guidelineBaseHeight = 680

const scale = (size: number) => (shortDimension / guidelineBaseWidth) * size
const verticalScale = (size: number) => (longDimension / guidelineBaseHeight) * size
const moderateScale = (size: number, factor = 0.5) => size + (scale(size) - size) * factor

export { scale, verticalScale, moderateScale }

const deviceHeight = isIphoneX() ? height - 78 : Platform.OS === 'android' ? height - StatusBar.currentHeight! : height

export function RFPercentage(percent: number) {
  const heightPercent = (percent * deviceHeight) / 100
  return Math.round(heightPercent)
}

export function RFValue(fontSize: number) {
  const standardScreenHeight = 680
  const heightPercent = (fontSize * deviceHeight) / standardScreenHeight
  return Math.round(heightPercent)
}

export const Text = styled.Text<TypographyProps & ColorProps & SpaceProps>(color, space, typography)

Text.defaultProps = {
  fontSize: RFValue(16),
  color: 'grayAlt',
  fontFamily: 'ProximaNova-Regular',
}

export const TextInput = styled.TextInput<
  TypographyProps & ColorProps & SpaceProps & BordersProps & FlexboxProps & LayoutProps
>(color, space, typography, borders, flexbox, layout)

TextInput.defaultProps = {
  fontSize: RFValue(16),
  color: 'grayAlt',
  fontFamily: 'ProximaNova-Regular',
}

export const SAV = styled.SafeAreaView<SpaceProps & LayoutProps & FlexboxProps & BordersProps & ColorProps>(
  space,
  layout,
  flexbox,
  color,
  borders
)

SAV.defaultProps = {
  flex: 1,
}

export const ScrollView = styled.ScrollView<SpaceProps & LayoutProps & FlexboxProps & BordersProps & ColorProps>(
  space,
  layout,
  flexbox,
  color,
  borders
)

ScrollView.defaultProps = {}

export const Box = styled.View<SpaceProps & LayoutProps & FlexboxProps & BordersProps & ColorProps & PositionProps>(
  space,
  position,
  layout,
  flexbox,
  color,
  borders
)

Box.defaultProps = {
  borderColor: 'whiteAlt',
}

export const BoxFill = styled(Box)`
  width: 100%;
`

export const BoxFullFill = styled(BoxFill)`
  height: 100%;
`

export const BoxRow = styled(Box)`
  flex-direction: row;
`

export const Touchable = styled.TouchableOpacity<
  SpaceProps & LayoutProps & FlexboxProps & BordersProps & PositionProps & ColorProps
>(space, position, layout, flexbox, color, borders)

Touchable.defaultProps = {
  borderColor: '#EFF2F4',
}

export const KeyboardAvoidingView = styled.KeyboardAvoidingView<SpaceProps & LayoutProps & FlexboxProps>(
  space,
  layout,
  flexbox
)

KeyboardAvoidingView.defaultProps = {
  flex: 1,
  behavior: 'padding',
}

export const H1 = styled.Text<PositionProps & TypographyProps & SpaceProps & ColorProps & FlexboxProps>(
  position,
  typography,
  space,
  color,
  flexbox
)

H1.defaultProps = {
  color: 'gray',
  fontWeight: 400,
  fontSize: RFValue(34),
  textAlign: 'center',
  fontFamily: 'ProximaNova-Regular',
}

export const H2 = styled(H1)`
  font-size: 28px;
`

export const H3 = styled(H1)`
  font-size: 24px;
`

export const H4 = styled(H1)`
  font-size: 22px;
`

export const H5 = styled(H1)`
  font-size: 20px;
`

export const H6 = styled(H1)`
  font-size: 18px;
`

export const H7 = styled(H1)`
  font-size: 16px;
`

export const Image = styled.Image<SpaceProps & PositionProps & FlexboxProps>(space, position, flexbox)

type Props = {
  source: any
}

export const FastImage: React.FC<Props & SpaceProps & LayoutProps & BordersProps & FastImageProperties> = props => {
  // TODO:! fix me
  //@ts-ignore
  return <RNFastImage style={{ ...props }} source={props.source} {...props} />
}
```
