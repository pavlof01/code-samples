# Code samples of B2B mobile project

### branch validation script
```js
const { exec } = require("child_process");
const chalk = require("chalk");

const branch_pattern = /^((features|fixes)|users\/[a-zA-z._-]*)\/b2b-\d{1,4}-[a-zA-Z0-9._-]+$/;
const exclude_branches = ["dev", "main"];

const errorMessage = chalk.redBright(
  `your branch name is not valid for pattern: \n ${branch_pattern} \n Please rename your branch (ex: users/username/b2b-1111-description-more-and-more)`,
);

function validateBranchName(branchName) {
  if (!branch_pattern.exec(branchName.trim())) {
    throw Error(errorMessage);
  }
}

exec("git rev-parse --abbrev-ref HEAD", (error, stdout, stderr) => {
  if (error) return console.log(chalk.red(`error: ${error.message}`));
  if (stderr) return console.log(chalk.red(`stderr: ${stderr}`));
  if (!exclude_branches.includes(stdout.trim())) validateBranchName(stdout);
});

exports.validateBranchName = validateBranchName;
exports.errorMessage = errorMessage;
```
### Tests for branch validation script
```js
/* eslint-env jest */
import { validateBranchName, errorMessage } from "./validate_branch_name";

describe("Test functions for validate name of current git branch name", () => {
  const validBranches = [
    "users/t-test/b2b-2-description-more-and-more",
    "users/t.test/b2b-20-description-more-and-more",
    "users/t_test/b2b-201-description-more-and-more",
    "users/T_test/b2b-2012-description-more-and-more",
    "users/T-TEST/b2b-2012-description-more-and-more",
    "users/LONG-LONG-text-LONG-LONG-text-LONG-LONG-text-LONG-LONG-text-LONG-LONG-text/b2b-2012-description-more-and-more",
    "features/b2b-1234-asdf-asdf",
    "fixes/b2b-1234-asdf-asdf",
  ];
  const invalidBranchesUserName = [
    "t!test/B2B-description-more-and-more",
    "t@test/B2B-12345-description-more-and-more",
    "t:test/B2B-12345-description-more-and-more",
    "t:test/sometext/B2B-12345-description-more-and-more",
    "t:test/some-text/B2B-12345-description-more-and-more",
  ];

  const invalidBranches = [
    "features/some-text/b2b-1234-asdf-asdf",
    "features/B2B-1234-asdf-asdf",
    "features/B2B-1234",
    "fixes/some-text/b2b-1234-asdf-asdf",
  ];

  const invalidBranchesJiraTicket = ["users/t-test/B2B-description-more-and-more", "users/t-test/B2B-12345-description-more-and-more"];

  test(`should throw Error with message ${errorMessage}`, () => {
    const branch = "branch-name-test";
    expect(() => validateBranchName(branch)).toThrow(errorMessage);
  });

  test.each(invalidBranchesJiraTicket)("should throw error because incorrect JIRA ticket (%p)", bName => {
    expect(() => validateBranchName(bName)).toThrow(errorMessage);
  });

  test.each(invalidBranchesUserName)("should throw error because incorrect USER name (%p)", bName => {
    expect(() => validateBranchName(bName)).toThrow(errorMessage);
  });

  test.each(invalidBranches)("should throw error because incorrect branch name (%p)", bName => {
    expect(() => validateBranchName(bName)).toThrow(errorMessage);
  });

  test.each(validBranches)("should successfully commit with branch name %p", bName => {
    expect(validateBranchName(bName)).toBeUndefined();
  });
});
```
### Create context util func
```js
import React, { useReducer, Dispatch, Reducer } from "react";
import { produce, Draft } from "immer";

/**
 * Action
 */
interface Action {
  type: string;
  payload?: any;
}

/**
 * Context dispatch
 */
interface ContextDispatch<Y> {
  dispatch: Dispatch<Y>;
}

/**
 * immer & reducer
 */
interface DraftReducer<X, Y> {
  (this: Draft<X>, draft: Draft<X>, action: Y): X;
}

/**
 * Context Provider
 * @param  initialState - State
 * @param  reducer      - immer & reducer
 */
function contextCreator<X extends Record<string, unknown>, Y extends Action>(initialState: X, reducer: DraftReducer<X, Y>) {
  // FIXME:
  // eslint-disable-next-line @typescript-eslint/ban-ts-comment
  // @ts-ignore
  const Context = React.createContext<X & ContextDispatch<Y>>(null);

  const ContextProvider: React.FC = ({ children }) => {
    const immerReducer = produce(reducer) as unknown as Reducer<X, Y>;
    const [state, dispatch] = useReducer(immerReducer, initialState);

    return <Context.Provider value={{ ...state, dispatch }}>{children}</Context.Provider>;
  };
  return { Context, ContextProvider };
}

export default contextCreator;
```
### Custom switch styles for MUI
```js
import { ComponentsOverrides, Theme } from "@mui/material";

export const customSwitch = (theme: Theme) =>
  ({
    styleOverrides: {
      root: {
        width: 42,
        height: 26,
        padding: 0,
      },
      thumb: {
        width: 22,
        height: 22,
        backgroundColor: theme.palette.common.white,
        boxSizing: "border-box",
      },

      track: {
        borderRadius: 26 / 2,
        backgroundColor: theme.palette.grey[500],
        opacity: 1,
        transition: theme.transitions.create(["background-color"], {
          duration: 500,
        }),
      },

      switchBase: {
        padding: 0,
        margin: 2,
        transitionDuration: "300ms",
        "&.Mui-checked": {
          transform: "translateX(16px)",
          color: "#fff",
          "& + .MuiSwitch-track": {
            backgroundColor: theme.palette.primary.main,
            opacity: 1,
            border: 0,
          },
          "&.Mui-disabled + .MuiSwitch-track": {
            opacity: 0.5,
          },
        },
        "&.Mui-focusVisible .MuiSwitch-thumb": {
          color: "#33cf4d",
          border: "6px solid #fff",
        },
        "&.Mui-disabled .MuiSwitch-thumb": {
          color: theme.palette.mode === "light" ? theme.palette.grey[100] : theme.palette.grey[600],
        },
        "&.Mui-disabled + .MuiSwitch-track": {
          opacity: theme.palette.mode === "light" ? 0.7 : 0.3,
        },
      },
    },
  } as {
    styleOverrides?: ComponentsOverrides["MuiSwitch"];
  });
```
### Header
```js
import { FC, memo } from "react";
import { Stack, Box } from "@mui/material";
import ReorderRoundedIcon from "@mui/icons-material/ReorderRounded";
import ShoppingCartOutlined from "@mui/icons-material/ShoppingCartOutlined";

import { ReactComponent as Logo } from "assets/icons/logo.svg";

type Props = {
  openDrawer: () => void;
};

const Header: FC<Props> = ({ openDrawer }) => {
  return (
    <Stack direction="row" justifyContent="space-between" py={4}>
      <Box component="div" onClick={openDrawer}>
        <ReorderRoundedIcon color="primary" />
      </Box>
      <Logo />
      <ShoppingCartOutlined color="secondary" />
    </Stack>
  );
};

export default memo(Header);
```
### Header.test
```js
import { render, fireEvent } from "@testing-library/react";

import Header from "./Header";

test("should render correctrly", () => {
  render(<Header openDrawer={jest.fn()} />);
});

test("should click on drawer menu icon for open", () => {
  const mockOnClick = jest.fn();
  const { getByTestId } = render(<Header openDrawer={mockOnClick} />);

  fireEvent.click(getByTestId("ReorderRoundedIcon"));

  expect(mockOnClick).toHaveBeenCalledTimes(1);
});

// TODO: need to write test with cypress
test.todo("write more tests for Header, HeaderLeft, HeaderRight, and other components if needed");
```
### Main
```js
import { useState } from "react";
import { TextField, Stack, Typography } from "@mui/material";

import { type ICatalogItem } from "containers/Catalog/types";

import CatalogModal from "containers/Catalog/components/CatalogModal";
import Header from "containers/Header";
import CatalogRootTemporary from "containers/Catalog/CatalogRootTemporary";
import MenuDrawer from "./components/MenuDrawer";
import Footer from "./components/Footer";
import SearchDialog from "./components/SearchDialog";

const Main = () => {
  const [isOpenSearchDialog, setIsOpenSearchDialog] = useState(false);
  const [isDrawerOpen, setIsDrawerOpen] = useState(false);
  const [currentViewingCatalogModal, setCurrentViewingCatalogModal] = useState<ICatalogItem | ICatalogItem[]>();

  const toggleDrawer = () => setIsDrawerOpen(!isDrawerOpen);

  const handleCloseCatalogDialog = () => setCurrentViewingCatalogModal(undefined);

  const handleClickOpen = () => setIsOpenSearchDialog(true);

  const handleClose = () => setIsOpenSearchDialog(false);

  return (
    <Stack>
      <Stack px={4}>
        <Header openDrawer={toggleDrawer} />
        <TextField label="Поиск товаров" variant="outlined" onClick={handleClickOpen} disabled />
        <Typography variant="h1" py={4}>
          Каталог
        </Typography>
        <CatalogRootTemporary setCurrentViewingCatalog={setCurrentViewingCatalogModal} />
      </Stack>
      <MenuDrawer isOpen={isDrawerOpen} toggleDrawer={toggleDrawer} setCurrentViewingCatalog={setCurrentViewingCatalogModal} />
      <SearchDialog isOpen={isOpenSearchDialog} handleClose={handleClose} />
      <CatalogModal
        equipment={currentViewingCatalogModal}
        isOpen={!(currentViewingCatalogModal == null)}
        onClose={handleCloseCatalogDialog}
        setCurrentViewingCatalog={setCurrentViewingCatalogModal}
      />
      <Footer />
    </Stack>
  );
};

export default Main;
```
### Main.test
```js
import { render } from "@testing-library/react";

import Main from "./Main";

test("should render correctrly", () => {
  render(<Main />);
});

test("match text Title and Footer", () => {
  const { getByText } = render(<Main />);

  expect(getByText("Каталог")).toBeInTheDocument();
});
```
### Menu drawer
```js
import { FC } from "react";
import {
  Stack,
  Typography,
  SwipeableDrawer,
  Avatar,
  List,
  ListItemButton,
  ListItemIcon,
  ListItemText,
  Divider,
  Button,
} from "@mui/material";

import RubleIcon from "@mui/icons-material/CurrencyRuble";
import DollarIcon from "@mui/icons-material/AttachMoney";
import EuroIcon from "@mui/icons-material/Euro";
import ManageSearchRoundedIcon from "@mui/icons-material/ManageSearchRounded";
import ShoppingCartOutlined from "@mui/icons-material/ShoppingCartOutlined";
import SearchRoundedIcon from "@mui/icons-material/SearchRounded";
import MailOutlinedIcon from "@mui/icons-material/MailOutlined";
import LogoutRoundedIcon from "@mui/icons-material/LogoutRounded";

import { type ICatalogItem } from "containers/Catalog/types";
import { iOS } from "libs/utils/common";

import User_avatar from "assets/images/user_avatar.jpg";
import User_avatar2x from "assets/images/user_avatar@2x.jpg";
import User_avatar3x from "assets/images/user_avatar@3x.jpg";
import User_avatar4x from "assets/images/user_avatar@4x.jpg";

import User_avatar2 from "assets/images/user_avatar_2.jpg";
import User_avatar2_2x from "assets/images/user_avatar_2@2x.jpg";
import User_avatar2_3x from "assets/images/user_avatar_2@3x.jpg";
import User_avatar2_4x from "assets/images/user_avatar_2@4x.jpg";

import CurrencyAccount from "./CurrencyAccount";
import ListItem from "./ListItem";

import listCategory from "containers/Catalog/mockData";

type Props = {
  isOpen: boolean;
  toggleDrawer: () => void;
  setCurrentViewingCatalog: (e: ICatalogItem | ICatalogItem[]) => void;
};

const MenuDrawer: FC<Props> = ({ isOpen, toggleDrawer, setCurrentViewingCatalog }) => {
  const LogOutButton = () => (
    <ListItemButton disableGutters sx={{ flexGrow: 0 }}>
      <ListItemIcon sx={{ minWidth: 36 }}>
        <LogoutRoundedIcon color="primary" />
      </ListItemIcon>
      <ListItemText primary="Выйти из аккаунта" primaryTypographyProps={{ variant: "body" }} />
    </ListItemButton>
  );

  const handleOpenRootCatalog = () => setCurrentViewingCatalog(listCategory.list);

  return (
    <SwipeableDrawer
      anchor="left"
      open={isOpen}
      onOpen={() => console.log("")}
      onClose={toggleDrawer}
      disableBackdropTransition={!iOS}
      disableDiscovery={iOS}
    >
      <Stack width="80vw" bgcolor="white" height="100vh">
        <Stack p={4} height="100%">
          <Avatar
            sx={{ width: 64, height: 64 }}
            src={User_avatar}
            srcSet={`${User_avatar2x} 2x, ${User_avatar3x} 3x, ${User_avatar4x} 4x`}
          />
          <Typography variant="h2" pt={4}>
            Имя пользователя
          </Typography>
          <Typography variant="subtitle1" color="text.secondary">
            Логин пользователя
          </Typography>
          <Stack bgcolor="grey.400" p={3} borderRadius={1} spacing={2} mt={4} mb={8}>
            <CurrencyAccount title="200 200 200,11" Icon={RubleIcon} />
            <CurrencyAccount title="600 200,32" Icon={DollarIcon} />
            <CurrencyAccount title="545 124,95" Icon={EuroIcon} />
          </Stack>
          <Stack>
            <List>
              <ListItem title="Каталог" Icon={ManageSearchRoundedIcon} onClick={handleOpenRootCatalog} />
              <ListItem title="Корзина" Icon={ShoppingCartOutlined} onClick={() => console.log("Корзина")} />
              <ListItem title="Найти" Icon={SearchRoundedIcon} onClick={() => console.log("Найти")} />
            </List>
          </Stack>
          <Stack justifyContent="flex-end" flex={1}>
            <Stack direction="row" alignItems="center" pb={4}>
              <Avatar
                sx={{ width: 64, height: 64 }}
                src={User_avatar2}
                srcSet={`${User_avatar2_2x} 2x, ${User_avatar2_3x} 3x, ${User_avatar2_4x} 4x`}
              />
              <Stack pl={4}>
                <Typography variant="subtitle3" color="text.secondary">
                  Менеджер по продажам
                </Typography>
                <Typography variant="h5" pt={1}>
                  Панфилова Наталья
                </Typography>
              </Stack>
            </Stack>
            <Button variant="contained" color="secondary" size="large" startIcon={<MailOutlinedIcon />}>
              Написать письмо
            </Button>
            <Divider sx={{ pb: 2, pt: 6 }} />
            <LogOutButton />
          </Stack>
        </Stack>
      </Stack>
    </SwipeableDrawer>
  );
};

export default MenuDrawer;
```
### Menu drawer.test
```js
import { render } from "@testing-library/react";

import MenuDrawer from "./MenuDrawer";

test("Should contains text", () => {
  const mockOnClick = jest.fn();
  const { getByText } = render(<MenuDrawer isOpen={true} toggleDrawer={mockOnClick} setCurrentViewingCatalog={mockOnClick} />);

  expect(getByText("Каталог")).toBeInTheDocument();
  expect(getByText("Корзина")).toBeInTheDocument();
  expect(getByText("Найти")).toBeInTheDocument();

  expect(getByText("Менеджер по продажам")).toBeInTheDocument();
  expect(getByText("Панфилова Наталья")).toBeInTheDocument();
  expect(getByText("Написать письмо")).toBeInTheDocument();

  expect(getByText("Выйти из аккаунта")).toBeInTheDocument();
});
```
