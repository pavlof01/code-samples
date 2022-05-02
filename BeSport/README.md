# Code samples of BeSport project

### Store root
```js
import { configureStore, combineReducers, createListenerMiddleware, applyMiddleware, isAnyOf, Middleware } from "@reduxjs/toolkit";
import { useSelector as useReduxSelector, TypedUseSelectorHook, useDispatch as useReduxDispatch } from "react-redux";
import createDebugger from "redux-flipper";

import { User } from "models";
import { getUsersDataById } from "firebase/utils";
import profile, { setFriends, setReceivedFriendRequests, setSentFriendRequests } from "./profile";
import chats from "./chats";
import createGame from "./createGame";
import fields from "./fields";
import football from "./football";
import users, { usersAddMany } from "./firestore/users";
import config from "./config";

const listenerMiddleware = createListenerMiddleware();

listenerMiddleware.startListening({
  matcher: isAnyOf(setFriends, setReceivedFriendRequests, setSentFriendRequests),
  effect: async (action, listenerApi) => {
    const users: User[] = await getUsersDataById(action.payload);
    listenerApi.dispatch(usersAddMany(users));
  },
});

const reducer = combineReducers({
  profile,
  createGame,
  fields,
  football,
  users,
  chats,
  config,
});

const middleware: Middleware[] = [];

if (__DEV__) {
  const reduxFlipperDebugger = createDebugger({ resolveCyclic: true });
  middleware.push(reduxFlipperDebugger);
}

const reduxFlipperDebugger = createDebugger({ resolveCyclic: true });
const store = configureStore({
  reducer,
  middleware: (getDefaultMiddleware) => getDefaultMiddleware({ serializableCheck: false }).prepend(listenerMiddleware.middleware),
  enhancers: [applyMiddleware(reduxFlipperDebugger)],
});

export type AppState = ReturnType<typeof store.getState>;

export const useSelector: TypedUseSelectorHook<AppState> = useReduxSelector;

export const useDispatch = () => useReduxDispatch();

export default store;

```
### Users store
```js
/* eslint-disable @typescript-eslint/unbound-method */
import { createSlice, createEntityAdapter, Action } from "@reduxjs/toolkit";
import { ThunkAction } from "redux-thunk";

import { fbRefPaths } from "firebase";
import { User } from "models";
import { AppState } from "store";

const usersAdapter = createEntityAdapter<User>({
  selectId: (user) => user.uid,
});

const usersSlice = createSlice({
  name: "users",
  initialState: usersAdapter.getInitialState(),
  reducers: {
    usersAddOne: usersAdapter.addOne,
    usersAddMany: usersAdapter.addMany,
    userUpdate: usersAdapter.updateOne,
    userRemove: usersAdapter.removeOne,
  },
});

export function loadMissingUsers(usersIds: string[]): ThunkAction<void, AppState, null, Action<string>> {
  return async (dispatch, getState) => {
    for (const id of usersIds) {
      const isExist = getState().users.ids.includes(id);
      if (!isExist) {
        const userSnapshot = await fbRefPaths(id).user.get();
        dispatch(usersAddOne(userSnapshot.data() as User));
      }
    }
  };
}

export const { selectById: selectUserById, selectAll: selectAllUsers } = usersAdapter.getSelectors<AppState>((state) => state.users);

export const { usersAddOne, usersAddMany, userUpdate, userRemove } = usersSlice.actions;

export default usersSlice.reducer;

```
### Subscribers firebase store
```js
/* eslint-disable @typescript-eslint/unbound-method */
import { createSlice, createEntityAdapter } from "@reduxjs/toolkit";

type Subscriber = { id: string; unsubscribe: () => void };

const subscribersAdapter = createEntityAdapter<Subscriber>({
  selectId: (user) => user.id,
});

const subscribersSlice = createSlice({
  name: "subscribers",
  initialState: subscribersAdapter.getInitialState(),
  reducers: {
    subscribersAddOne: subscribersAdapter.addOne,
    subscribersAddMany: subscribersAdapter.addMany,
    subscribersUpdate: subscribersAdapter.updateOne,
    subscribersRemove: subscribersAdapter.removeOne,
  },
});

export const { subscribersAddOne, subscribersAddMany, subscribersUpdate, subscribersRemove } = subscribersSlice.actions;

export default subscribersSlice.reducer;

```
### Profile actions
```js
import { AnyAction, ThunkAction } from "@reduxjs/toolkit";

import { auth, QuerySnapshot, arrayRemove, arrayUnion, fbUserRefs, fbRefPaths } from "firebase";
import { AppState } from "store";
import { User } from "models";
import { subscribersAddMany } from "store/firestore/subscribers";
import * as profileActions from "store/profile";
import { usersAddOne } from "store/firestore/users";
import { getFootballSnapshot } from "store/football/actions";
import { /* ChatsActions, */ onChatSubscribe } from "store/chats/actions";
import { getPrivateChatRoomId } from "utils";

import PushNotification from "services/PushNotifications";

export function subscribeOnUserChanges(): ThunkAction<void, AppState, null, AnyAction> {
  return (dispatch) => {
    dispatch(profileActions.setLoading(true));
    const unsubscribeUserData = fbRefPaths().user.onSnapshot(
      (snapshot) => {
        dispatch(profileActions.setProfile(snapshot.data() as User));
        dispatch(usersAddOne(snapshot.data() as User));
      },
      (err) => dispatch(profileActions.setError(err.toString())),
    );
    const unsubscribeReceivedFriendRequests = fbUserRefs().receivedFriendRequests.onSnapshot(
      (snapshot) => dispatch(profileActions.setReceivedFriendRequests(snapshot)),
      (err) => dispatch(profileActions.setError(err.toString())),
    );
    const unsubscribeSentFriendRequests = fbUserRefs().sentFriendRequests.onSnapshot(
      (snapshot) => dispatch(profileActions.setSentFriendRequests(snapshot)),
      (err) => dispatch(profileActions.setError(err.toString())),
    );
    const unsubscribeFriends = fbUserRefs().friends.onSnapshot(
      (snapshot) => {
        snapshot.docs.forEach(({ id }) => dispatch(onChatSubscribe(getPrivateChatRoomId(id))));
        dispatch(profileActions.setFriends(snapshot));
      },
      (err) => dispatch(profileActions.setError(err.toString())),
    );
    const unsubscribeFootballInvites = fbUserRefs().footballInvites.onSnapshot(
      (snapshot) => {
        dispatch(profileActions.setFootballInvites(snapshot));
        checkSnapshotInvites(snapshot);
      },
      (err) => dispatch(profileActions.setError(err.toString())),
    );
    dispatch(
      subscribersAddMany([
        { id: "subscribeUserData", unsubscribe: unsubscribeUserData },
        { id: "subscribeReceivedFriendRequests", unsubscribe: unsubscribeReceivedFriendRequests },
        { id: "subscribeSentFriendRequests", unsubscribe: unsubscribeSentFriendRequests },
        { id: "subscribeFriends", unsubscribe: unsubscribeFriends },
        { id: "subscribeFootballInvites", unsubscribe: unsubscribeFootballInvites },
      ]),
    );
    dispatch(profileActions.setLoading(false));
  };
}

// TODO: need to write common func for check exist snapshot

const checkSnapshotInvites = (snapshot: QuerySnapshot) => {
  if (!snapshot.empty) {
    snapshot.forEach(async (invite) => {
      const footballSnapshot = await getFootballSnapshot(invite.data()?.gameId);
      const keywords = footballSnapshot.data()?.keywords || [];
      const isInviteNotValid = keywords.includes("full") || keywords.includes("started") || keywords.includes("ended") || !footballSnapshot;
      if (isInviteNotValid || !footballSnapshot.exists) {
        await deleteFootballInvite(invite.id);
      }
    });
  }
};

const deleteFootballInvite = (id: string) => fbUserRefs().footballInvites.doc(id).delete();

export function setPushToken(pushtoken: string) {
  fbUserRefs().common.update({ pushtoken });
}

export function subscribedToTopic(topic: string): Promise<void> {
  return PushNotification.subscribeToTopic(topic).then(() => fbUserRefs().common.update({ subscribedTopics: arrayUnion(topic) }));
}

export function unsubscribedToTopic(topic: string): Promise<void> {
  return PushNotification.unsubscribeToTopic(topic).then(() => fbUserRefs().common.update({ subscribedTopics: arrayRemove(topic) }));
}

/* ------ ACTIONS FOR USER FRIENDS ------ */

export function sendFriendRequest(userId: string) {
  fbUserRefs()
    .sentFriendRequests.doc(userId)
    .set({ [userId]: userId });
  fbUserRefs(userId)
    .receivedFriendRequests.doc(auth.currentUser?.uid)
    .set({ [auth.currentUser!.uid]: auth.currentUser?.uid });
}

function addNewFriendIdDatabase(userId: string) {
  fbUserRefs()
    .friends.doc(userId)
    .set({ [userId]: userId });
  fbUserRefs(userId)
    .friends.doc(auth.currentUser?.uid)
    .set({ [auth.currentUser!.uid]: auth.currentUser?.uid });
}

export function acceptFriendRequest(uid: string) {
  fbUserRefs().receivedFriendRequests.doc(uid).delete();
  fbUserRefs(uid).sentFriendRequests.doc(auth.currentUser?.uid).delete();
  addNewFriendIdDatabase(uid);
}

export function deleteFriend(userId: string) {
  fbUserRefs().friends.doc(userId).delete();
  fbUserRefs(userId).friends.doc(auth.currentUser?.uid).delete();
}

```
### Profile store
```js
import { createSlice, PayloadAction } from "@reduxjs/toolkit";

import { AppState } from "store";

import { QuerySnapshot } from "/firebase";

import { User } from "models";

type IState = {
  isLoading: boolean;
  isLogin: boolean;
  user?: User;
  receivedFriendRequests?: QuerySnapshot;
  sentFriendRequests?: QuerySnapshot;
  footballInvites?: QuerySnapshot;
  friends?: QuerySnapshot;
  error?: string;
};

const initialState: IState = {
  isLoading: false,
  isLogin: false,
  user: undefined,
  receivedFriendRequests: undefined,
  sentFriendRequests: undefined,
  footballInvites: undefined,
  friends: undefined,
  error: undefined,
};

export const profileSlice = createSlice({
  name: "profile",
  initialState,
  reducers: {
    setLoading: (state, { payload }: PayloadAction<boolean>) => {
      state.isLoading = payload;
    },
    setProfile: (state, { payload }: PayloadAction<User>) => {
      state.isLogin = true;
      state.user = payload;
    },
    setFriends: (state, { payload }: PayloadAction<QuerySnapshot>) => {
      state.friends = payload;
    },
    setFootballInvites: (state, { payload }: PayloadAction<QuerySnapshot>) => {
      state.footballInvites = payload;
    },
    setReceivedFriendRequests: (state, { payload }: PayloadAction<QuerySnapshot>) => {
      state.receivedFriendRequests = payload;
    },
    setSentFriendRequests: (state, { payload }: PayloadAction<QuerySnapshot>) => {
      state.sentFriendRequests = payload;
    },
    setError: (state, { payload }: PayloadAction<string>) => {
      state.error = payload;
    },
    setLogOut: (state) => {
      state.isLogin = false;
      state.user = undefined;
    },
  },
});

export const {
  setLoading,
  setProfile,
  setFriends,
  setFootballInvites,
  setReceivedFriendRequests,
  setSentFriendRequests,
  setError,
  setLogOut,
} = profileSlice.actions;

export const authSelector = (state: AppState) => state;

export default profileSlice.reducer;

```
### Profile selectors
```js
import { createSelector } from "@reduxjs/toolkit";
import { isBefore } from "date-fns";
import { compact } from "lodash";

import { QuerySnapshot } from "firebase";
import { isGameEnded } from "store/selectors/football";
import { selectAll } from "store/football";
import { AppState } from "store";
import { User, FootballGame, Invite } from "models";

const currentUserUid = (state: AppState) => state.profile.user?.uid;
const profileState = (state: AppState) => state.profile;
const user = (state: AppState) => state.profile.user;
const footballInvitesSnapshot = (state: AppState) => state.profile.footballInvites as QuerySnapshot<Invite>;

const friendDocs = (state: AppState) => state.profile.friends?.docs;
const friendsIds = (state: AppState) => state.profile.friends?.docs.map((doc) => doc.id);
const friends = (state: AppState) =>
  state.profile.friends?.docs.map(({ id }) => {
    return state.users.entities[id];
  }, []);
const sentFriendRequestsDocs = (state: AppState) => state.profile.sentFriendRequests?.docs;
const sentFriendRequests = (state: AppState) =>
  state.profile.sentFriendRequests?.docs.map(({ id }) => {
    return state.users.entities[id];
  }, []);
const receivedFriendRequestsDocs = (state: AppState) => state.profile.receivedFriendRequests?.docs;
const receivedFriendRequests = (state: AppState) =>
  state.profile.receivedFriendRequests?.docs.map(({ id }) => {
    return state.users.entities[id];
  }, []);

export const isSubscribedToTopic = (user: User, topicId: string): boolean | undefined => user?.subscribedTopics?.includes(topicId);
const isUserJoined = (game: FootballGame, userUid?: string) => game.players.includes(userUid || "");

export const selectCurrentUserUid = createSelector(currentUserUid, (currentUserUid) => currentUserUid);

export const selectProfileState = createSelector(profileState, (state) => state);

export const selectUser = createSelector(user, (user) => user);

export const selectFriendsDocs = createSelector(friendDocs, (docs) => docs);
export const selectFriendsIds = createSelector(friendsIds, (ids) => ids);
export const selectFriends = createSelector(friends, (friends) => compact(friends));

export const selectReceivedFriendRequestsDocs = createSelector(receivedFriendRequestsDocs, (docs) => docs);
export const selectReceivedFriendRequests = createSelector(receivedFriendRequests, (receivedFriendRequests) =>
  compact(receivedFriendRequests),
);

export const selectSentFriendRequestsDocs = createSelector(sentFriendRequestsDocs, (docs) => docs);
export const selectSentFriendRequests = createSelector(sentFriendRequests, (sentFriendRequests) => compact(sentFriendRequests));

export const selectNextUserGame = createSelector(user, selectAll, (profile, footballGames) =>
  footballGames
    .filter((game) => profile?.upcomingGames?.includes(game.id) && !isGameEnded(game))
    .sort((a, b) => {
      return isBefore(new Date(a.date), new Date(b.date)) ? 1 : -1;
    }),
);

export const selectPastGames = createSelector(selectAll, selectCurrentUserUid, (games, uid) =>
  games?.filter((game) => isUserJoined(game, uid) && game.keywords.includes("ended")),
);

export const selectIsSubscribedToTopic = createSelector(isSubscribedToTopic, (res) => res);

export const selectFootballInvitesDocs = createSelector(footballInvitesSnapshot, (docs) => docs);

```
### Profile screen
```js
import React from "react";
import { SafeAreaView, ScrollView, View, Text } from "react-native";
import { useNavigation } from "@react-navigation/native";
import { useSelector } from "store";
import ContentLoader, { Rect, Circle } from "react-content-loader/native";

import translate from "utils/i18n";

import { Layout, Typography, Palette } from "styles";
import { User, FootballGame } from "models";
import {
  selectProfileState,
  selectNextUserGame,
  selectPastGames,
  selectFriends,
  selectReceivedFriendRequests,
} from "store/selectors/profile";

import Avatar from "components/Avatar";
import Button from "components/Button";
import PlayerCard from "components/PlayerCard";
import GameReminder from "components/GameReminder";
import FootballCard from "components/FootballCard";
import HorizontalList from "components/HorizontalList";

import { width } from "utils";

const Profile: React.FC = () => {
  const navigation = useNavigation();
  const profileState = useSelector(selectProfileState);
  const friends = useSelector(selectFriends);
  const receivedFriendRequests = useSelector(selectReceivedFriendRequests);
  const nextGame = useSelector(selectNextUserGame)[0];
  const pastGames = useSelector(selectPastGames);

  const user = profileState.user;

  if (profileState.isLoading) {
    const startY = 50 + 5;
    return (
      <ContentLoader
        height={1500}
        backgroundColor={Palette.background.paper}
        foregroundColor={Palette.background.default}
        speed={1.5}
        interval={0}
      >
        <Circle x={width / 2} y={startY} cx="4" cy="4" r={50} />
        <Rect x={5} y={startY + 80} rx="4" ry="4" width={width - 10} height="100" />
        <Rect x={5} y={250} rx="4" ry="4" width={120} height="140" />
        <Rect x={135} y={250} rx="4" ry="4" width={120} height="140" />
        <Rect x={135 + 120 + 10} y={250} rx="4" ry="4" width={120} height="140" />
      </ContentLoader>
    );
  }

  if (!profileState.isLogin)
    return (
      <SafeAreaView style={Layout.SafeAreaView}>
        <Button title={translate("profile.login_btn")} onPress={() => navigation.navigate("Login")} />
      </SafeAreaView>
    );

  const toFriendsRoute = () => navigation.navigate("Friends");

  const _renderUserItem = ({ item }: { item: User }) => <PlayerCard user={item} />;

  const _renderPastGameItem = ({ item }: { item: FootballGame }) => <FootballCard game={item} />;

  const keyExtractorForUser = (item: User) => item.uid;
  const keyExtractorForPastGame = (item: FootballGame) => item.id;

  return (
    <SafeAreaView style={Layout.SafeAreaView}>
      <ScrollView contentContainerStyle={Layout.SmallestGutterHorizontal} showsVerticalScrollIndicator={false}>
        <View style={Layout.Center}>
          <Avatar data={user} size="main" pressable={false} />
          <Text style={Typography.h3}>{user?.displayName}</Text>
          <Text style={Typography.subtitle2}>{user?.city?.name}</Text>
          <Text style={Typography.subtitle2}>{user?.uid}</Text>
        </View>
        {nextGame ? <GameReminder game={nextGame} /> : null}
        <View>
          <HorizontalList<User>
            title={translate("profile.friends.friend_requests")}
            data={receivedFriendRequests}
            renderItem={_renderUserItem}
            keyExtractor={keyExtractorForUser}
          />
          <HorizontalList<User>
            title={translate("common.friends")}
            onTitlePress={toFriendsRoute}
            data={friends}
            renderItem={_renderUserItem}
            keyExtractor={keyExtractorForUser}
          />
          <HorizontalList<FootballGame>
            title={translate("profile.past_games")}
            data={pastGames}
            renderItem={_renderPastGameItem}
            keyExtractor={keyExtractorForPastGame}
          />
        </View>
      </ScrollView>
    </SafeAreaView>
  );
};

export default Profile;

```
### Football field page
```js
import React, { useState } from "react";
import { View, ScrollView, FlatList, Text, TextInput, StyleSheet } from "react-native";
import RNFastImage from "react-native-fast-image";
import { useNavigation } from "@react-navigation/native";
import Carousel, { Pagination } from "react-native-snap-carousel";
import { RouteProp } from "@react-navigation/native";
import { MainStackStackParamList } from "stacks/MainStack";
import MapView, { Marker } from "react-native-maps";
import LinearGradient from "react-native-linear-gradient";
import { get } from "lodash";
import { useSelector, useDispatch } from "store";

import translate from "utils/i18n";

import { FootballGame } from "models";

import { setValue } from "store/createGame";
import { selectUser } from "store/selectors/profile";
import { selectGamesByFieldId } from "store/selectors/football";

import { Typography, Layout, Palette, screenWidth, screenHeight } from "styles";

import Button from "components/Button";
import { RowInfo } from "components/common";
import UpcomingGameCard from "components/UpcomingGameCard";
import Title from "/components/HorizontalList/Title";
import { ItemSeparatorComponentHorizontal } from "components/common";

const styles = StyleSheet.create({
  dot: {
    width: 10,
    height: 10,
    borderRadius: 5,
    marginHorizontal: 8,
    backgroundColor: Palette.common.white,
  },
  pagination: {
    bottom: 0,
    height: 40,
    position: "absolute",
    alignItems: "center",
  },
  linearGradientTop: {
    height: "10%",
    width: "100%",
    position: "absolute",
    top: 0,
    zIndex: 999,
  },
  linearGradientBottom: {
    height: "30%",
    width: screenWidth,
    position: "absolute",
    bottom: 0,
  },
  sectionTitle: {
    ...Typography.h3,
    ...Typography.withGutters,
  },
  screenItemBox: {
    position: "relative",
  },
  screenItemImageBackground: {
    height: screenHeight / 2,
  },
  mapView: {
    flex: 1,
    height: screenHeight / 3,
  },
});

type FootballGameScreenScreenRouteProp = RouteProp<MainStackStackParamList, "FootballFieldPage">;

type Props = {
  route: FootballGameScreenScreenRouteProp;
};

const FootballfieldDataPage: React.FC<Props> = ({ route }) => {
  const navigation = useNavigation();
  const dispatch = useDispatch();
  const profile = useSelector(selectUser);
  const games = useSelector((state) => selectGamesByFieldId(state, route.params.field.id));
  const [activeSlide, setActiveSlide] = useState(0);
  const field = route.params.field;

  const REGION = {
    latitude: field.latitude,
    longitude: field.longitude,
    latitudeDelta: 0.006,
    longitudeDelta: 0.0121,
  };

  const startCreateGame = () => {
    if (!profile?.uid) return navigation.navigate("Login");
    dispatch(setValue({ valueName: "field", value: field }));
    navigation.navigate("ChooseDate");
  };

  const renderScreen = ({ item }: { item: string }) => {
    return (
      <View style={styles.screenItemBox}>
        <RNFastImage style={styles.screenItemImageBackground} source={{ uri: item }}>
          <LinearGradient
            colors={["rgba(0,0,0,0)", "rgba(0,0,0,1)"]}
            start={{ x: 0.0, y: 0.1 }}
            end={{ x: 0.0, y: 0.85 }}
            style={styles.linearGradientBottom}
          />
        </RNFastImage>
      </View>
    );
  };

  const _renderGameItem = ({ item }: { item: FootballGame }) => <UpcomingGameCard game={item} hideFieldTitle hideFieldAddress />;

  const keyExtractorGame = (item: FootballGame) => item.id;

  const ImageCarousel = () => (
    <View>
      <Carousel
        sliderWidth={screenWidth}
        itemWidth={screenWidth}
        data={field?.images || [field.coverImage]}
        renderItem={renderScreen}
        onSnapToItem={(index) => setActiveSlide(index)}
        inactiveSlideOpacity={0.8}
        inactiveSlideScale={1}
      />
      {get(field, "images", []).length > 1 && (
        <View style={styles.pagination}>
          <Pagination
            dotsLength={field.images?.length || 0}
            activeDotIndex={activeSlide}
            dotStyle={styles.dot}
            containerStyle={{ width: "25%", justifyContent: "center" }}
          />
        </View>
      )}
    </View>
  );

  return (
    <View style={Layout.DefaultBackgroundColor}>
      <ScrollView>
        {ImageCarousel()}
        <View style={Layout.SmallerGutterHorizontal}>
          <View style={Layout.RowAlignBottom}>
            <Text style={Typography.h3}>{field.name},</Text>
            <Text style={Typography.body2}> {field.address}</Text>
          </View>
          <MapView style={styles.mapView} region={REGION}>
            <Marker coordinate={{ latitude: field.latitude, longitude: field.longitude }} title={field.name} description={field.address} />
          </MapView>
          <View style={Layout.SmallestGutterVertical}>
            <Text style={styles.sectionTitle}>{translate("football_field_page.field_info_title")}</Text>
            <RowInfo title={translate("football_game_page.game_settings.game_format")} value={`${22 / 2}/${22 / 2}`} />
            <RowInfo title={translate("football_game_page.game_settings.field_size")} value="Standart" />
            <RowInfo title={translate("football_game_page.game_settings.field_coverage")} value="Artificial turf" />
            <RowInfo title={translate("common.price")} value={`${field.price} rub/hour`} />
          </View>
          <View style={Layout.SmallestGutterVertical}>
            <Text style={styles.sectionTitle}>{translate("football_game_page.game_settings.additionally")}</Text>
            <TextInput style={Typography.body2} editable={false} value="description" multiline />
          </View>
          {games.length ? (
            <View style={Layout.SmallGutterVertical}>
              <Title title={translate("football_field_page.game_on_fields_title")} />
              <FlatList
                data={games}
                ListEmptyComponent={null}
                renderItem={_renderGameItem}
                keyExtractor={keyExtractorGame}
                ItemSeparatorComponent={ItemSeparatorComponentHorizontal}
                horizontal
                showsHorizontalScrollIndicator={false}
              />
            </View>
          ) : null}
          <Button title={translate("football_field_page.create_game_on_this_field")} onPress={startCreateGame} />
        </View>
      </ScrollView>
      <LinearGradient
        style={styles.linearGradientTop}
        colors={["rgba(0,0,0,1)", "rgba(0,0,0,0)"]}
        start={{ x: 0.0, y: 0.0 }}
        end={{ x: 0.0, y: 0.99 }}
      />
    </View>
  );
};

export default FootballfieldDataPage;

```
### Football card
```js
import React from "react";
import { StyleSheet, Image, Pressable, View, Text } from "react-native";
import RNFastImage from "react-native-fast-image";
import LinearGradient from "react-native-linear-gradient";
import { useNavigation } from "@react-navigation/native";
import { values } from "lodash";
import moment from "moment";

import { FootballGame } from "models";
import { Typography } from "styles";

const styles = StyleSheet.create({
  contentContainer: {
    width: 250,
    height: 250,
  },
  body: {
    justifyContent: "space-between",
    padding: 6,
  },
  bodyHeader: {
    flexDirection: "row",
    alignItems: "center",
  },
  footerContainer: {
    flexDirection: "row",
    justifyContent: "space-between",
  },
  iconContainer: {
    alignItems: "center",
    flexDirection: "row",
  },
  icon: {
    marginRight: 5,
  },
  linearGradient: {
    height: "70%",
    width: 250,
    position: "absolute",
    bottom: 0,
    borderRadius: 6,
  },
  imageContainer: {
    ...StyleSheet.absoluteFillObject,
    resizeMode: "cover",
    justifyContent: "flex-end",
    borderRadius: 6,
  },
});

type Props = {
  game: FootballGame;
};

const FootballCard: React.FC<Props> = ({ game }) => {
  const navigation = useNavigation();
  const { id, date, maxPlayers, field, price, players, duration } = game;
  const currentNumPLayersInGame = values(players).length;
  const toGameRoute = () => navigation.navigate("FootballGamePage", { id, game });
  return (
    <Pressable onPress={toGameRoute}>
      <View style={styles.contentContainer}>
        <RNFastImage source={{ uri: field?.coverImage }} style={styles.imageContainer}>
          <LinearGradient
            colors={["rgba(0,0,0,0)", "rgba(0,0,0,1)"]}
            start={{ x: 0.0, y: 0.1 }}
            end={{ x: 0.0, y: 0.85 }}
            style={styles.linearGradient}
          />
          <View style={styles.body}>
            <Text style={Typography.paragraph}>{field?.name}</Text>
            <Text style={Typography.subtitle2}>{field?.address}</Text>
            <View style={styles.footerContainer}>
              <View style={styles.iconContainer}>
                <Image style={styles.icon} source={require("assets/icons/peoples.png")} />
                <Text style={Typography.paragraph}>
                  <Text>{currentNumPLayersInGame}</Text>/{maxPlayers}
                </Text>
              </View>
              <View style={{ alignItems: "center" }}>
                <View style={styles.bodyHeader}>
                  <Image style={styles.icon} source={require("assets/icons/clock.png")} />
                  <Text style={Typography.subtitle1}>{moment(date).format("DD MMM HH:mm")}</Text>
                </View>
                <Text style={Typography.subtitle1}>{duration} min</Text>
              </View>
              <View style={styles.iconContainer}>
                <Image style={styles.icon} source={require("assets/icons/wallet.png")} />
                <Text style={Typography.paragraph}>{price} rub</Text>
              </View>
            </View>
          </View>
        </RNFastImage>
      </View>
    </Pressable>
  );
};

export default FootballCard;

```
