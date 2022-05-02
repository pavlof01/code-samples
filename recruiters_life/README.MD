# Code samples of Recruiters Life project

### API
```js
export const API_BASE_URL = process.env.REACT_APP_API_SERVER_URL

class Api {
  static tokenName = 'authToken'

  private static instance: Api

  private static authToken: string

  axiosInstance: AxiosInstance

  constructor() {
    this.axiosInstance = axios.create({
      baseURL: API_BASE_URL,
      headers: {
        Accept: 'application/json',
        'Content-Type': 'application/json',
      },
    })

    this.axiosInstance.interceptors.response.use(
      response => response,
      error => {
        if (error.response) {
          if (error.response.status === 403) {
            return localStorage.removeItem('rlToken')
          } else {
            return Promise.reject(error)
          }
        }
      }
    )
  }

  static getInstance() {
    if (!Api.instance) {
      Api.instance = new Api()
    }
    return Api.instance
  }

  static getAxios() {
    return Api.getInstance().axiosInstance
  }

  /**
   * SET token of auth
   *
   * @param {string} token
   */

  static setAuthToken(token: string) {
    this.authToken = token
    Api.getInstance().axiosInstance.defaults.headers.common.Authorization = `Bearer ${this.authToken}`
  }

  static getAuthToken() {
    return this.authToken
  }

  /**
   * GET
   *
   * @param {string} url
   * @param {object} params
   * @return {AxiosPromise}
   */

  static get<T = any>(url: string, config?: AxiosRequestConfig): AxiosPromise<T> {
    return Api.getAxios().get(url, config)
  }

  /**
   * POST
   *
   * @param {string} url
   * @param {any} data
   * @param {AxiosRequestConfig} config
   * @return {AxiosPromise}
   */

  static post<T = any>(url: string, data?: any, config?: AxiosRequestConfig): AxiosPromise<T> {
    return Api.getAxios().post(url, data, config)
  }

  static patch<T = any>(url: string, data?: any, config?: AxiosRequestConfig): AxiosPromise<T> {
    return Api.getAxios().patch(url, data, config)
  }

  static delete<T = any>(url: string, config?: AxiosRequestConfig): AxiosPromise<T> {
    return Api.getAxios().delete(url, config)
  }
}

export default Api
```
### API OF EMPLOYERS
```js
import Api, { API_BASE_URL } from './index'
import { IEmployer, IJobPosition, IEmployerContact } from '../types/models'
import {
  IDataEmployersLiveJobs,
  IDataEmployerWithPagedJobPositions,
  IDataJobPositionWithPagedPotentialCandidates,
} from '../types/responseData'
import { IQueryParams } from '../types/helps'
import { IUploadInputParams } from '../components/UploadInput/types'

const EMPLOYER_BASE_URL = `${API_BASE_URL}/user/employers`

class Employers {
  /**
   * Gets employers with recent job position for Live Jobs screen
   */
  static getLiveJobs(query: IQueryParams) {
    return Api.get<IDataEmployersLiveJobs>('/user/employers/live-jobs/', { params: query })
  }

  /**
   * Gets employers with recent job position for Live Jobs screen
   */
  static getEmployers(id: number, page?: number, perPage?: number) {
    return Api.get<IDataEmployerWithPagedJobPositions>(`/user/employers/${id}/with-paged-job-positions/`, {
      params: { page, perPage },
    })
  }

  /**
   * Create new employer
   */
  static createEmployer(employer: Partial<IEmployer>) {
    return Api.post<IEmployer>('/user/employers/', employer)
  }

  /**
   * Create new contact for employer
   */
  static createEmployerContact(employerId: number, contact: Partial<IEmployerContact>) {
    return Api.post<IEmployerContact>(`/user/employers/${employerId}/contacts/`, contact)
  }

  /**
   * Edit contact for employer
   */
  static editEmployerContact(contactId: number, contact: Partial<IEmployerContact>) {
    return Api.patch<IEmployerContact>(`/user/employers/contacts/${contactId}/`, contact)
  }

  /**
   * Deletes employer contact with required contact id
   */
  static deleteEmployerContact(contactId: number) {
    return Api.delete(`/user/employers/contacts/${contactId}/`)
  }

  /**
   * Updates employer with required id
   */
  static updateEmployer(id: number, employer: Partial<IEmployer>) {
    return Api.patch(`/user/employers/${id}/`, employer)
  }

  /**
   * Deletes employer with required id
   */
  static deleteEmployer(id: number) {
    return Api.delete(`/user/employers/${id}/`)
  }

  /**
   * Create job position for chosen employer
   */
  static createJobPosition(employerId: number, jobPosition: Partial<IJobPosition>) {
    return Api.post<IJobPosition>(`/user/employers/${employerId}/job-positions/`, jobPosition)
  }

  /**
   * Gets job position with required id
   */
  static getJobPosition(jobPositionId: number) {
    return Api.get<IJobPosition>(`/user/employers/job-positions/${jobPositionId}`)
  }

  /**
   * --user/employers/job-positions/{id}/with-paged-potential-candidates/--
   * Gets selected job position with paged potential candidates
   */
  static getJobPositionWithPotentialCandidates(jobPositionId: number, page?: number, perPage?: number) {
    return Api.get<IDataJobPositionWithPagedPotentialCandidates>(
      `/user/employers/job-positions/${jobPositionId}/with-paged-potential-candidates/`,
      {
        params: { page, perPage },
      }
    )
  }

  /**
   * Updates job position with required id if it belongs to user
   */
  static updateJobPosition(id: number, job: Partial<IJobPosition>) {
    return Api.patch(`/user/employers/job-positions/${id}/`, job)
  }

  static setStatusJobPosition(id: number, value: boolean) {
    return Api.patch<IJobPosition>(`/user/employers/job-positions/${id}/`, { closed: value })
  }

  /**
   * Deletes job position with required id if it belongs to user
   */
  static deleteJobPosition(id: number) {
    return Api.delete(`/user/employers/job-positions/${id}/`)
  }
}

export async function paramsForUploadLogoEmployer(employerId: number): Promise<IUploadInputParams> {
  // const id = candidateId || (await getDraftCandidateId())
  return {
    fieldName: 'logoFile',
    options: {
      url: `${EMPLOYER_BASE_URL}/${employerId}/files/logo/`,
      headers: { Authorization: `Bearer ${Api.getAuthToken()}` },
    },
  }
}

export default Employers
```
### Form validation with yup
```js
import _ from 'lodash'
import * as yup from 'yup'
import { ValidationError } from 'yup'
import UserApi from '../api/auth'

/* func for format errors object to key:error */
export const formatErrors = (err: ValidationError): object => {
  if (!err.inner) return {}
  const errors: any = {}
  err.inner.forEach((error: ValidationError) => {
    errors[error.path] = error.errors[0]
  })
  return errors
}

/* func for format errors object to key:error for server response */
export const formatErrorsForResponse = (errs: any[]) => {
  const errors: any = {}
  errs.forEach((err: any) => {
    errors[err.fieldName] = err!.errorMessage
  })
  return errors
}

/* ------------------------------------------- */

/**
 * check is email already exist
 * @param {message} - error message
 * @param {isExist} - revert function.
 *                    For default func return error if email exist.
 *                    If isExist = false, return no error
 */

function isEmailExist(this: yup.StringSchema, message: string = 'this email is already taken', isExist = true) {
  return this.test('unique-email', message, value =>
    value
      ? UserApi.isEmailExist(value)
          .then(({ data }) => (isExist ? !data.emailExist : data.emailExist))
          .catch(() => true)
      : true
  )
}

yup.addMethod(yup.string, 'isEmailExist', isEmailExist)

/* ------------------------------------------- */

const Errors = {
  notValid: 'Not valid',
  notEmpty: 'Must be non empty',
  onlyLatinSymbols: 'Use only latin symbols',
  onlyNumbers: 'Use only numbers',
  notValidEmail: 'Email not valid',
  onlyNumbersAndLatinSymbols: 'Use only latin symbols and numbers',
  notValidLinkedIn: 'LinkedIn profile not valid',
  passwordError: 'Password must contain lower and upper case character',
}

const textPattern = /^[a-zA-Z\s]+$/
const numericPattern = /^[0-9]+$/
const numericPatternWithSpaces = /^(?=.*\d)[\d ]+$/
const textNumericWithSpecialCharactersPattern = /^[0-9a-zA-Z\s,.\/\\]+$/
// const linkedInPattern = /^(http(s)?:\/\/)?([\w]+\.)?linkedin\.com\/(pub|in|profile)/
const emailPattern = /^([a-zA-Z0-9_\.-]+\@[\da-z\.-]+\.[a-z\.]{2,6})$/
// const registrationNamePattern = /^[A-Z][a-zA-Z]*$/
const passwordPattern = /^((?=.*[a-z])(?=.*[A-Z]).{6,255})$/

const textSchema = yup
  .string()
  .trim()
  .matches(textPattern, { message: Errors.onlyLatinSymbols, excludeEmptyString: true })
  .max(50)
  .required()

const regName = yup
  .string()
  .trim()
  .test('checkNumbers', 'No numbers', (val: string) => !/\d/.test(val))
  .max(50)
  .min(1)
  .required()

const textNumericSchema = yup
  .string()
  .max(50)
  .required()

const numericSchema = yup.string().matches(numericPattern, Errors.onlyNumbers)

const numericSchemaWithSpaces = yup
  .string()
  .matches(numericPatternWithSpaces, { message: Errors.onlyNumbers, excludeEmptyString: true })

const emailSchema = yup
  .string()
  .matches(emailPattern, Errors.notValidEmail)
  .email(Errors.notValidEmail)
  .required()

const phoneSchema = yup.string().matches(numericPattern, { message: Errors.onlyNumbers, excludeEmptyString: true })

const linkedInSchema = yup.string().label('linkedIn')
// .matches(linkedInPattern, { message: Errors.notValidLinkedIn, excludeEmptyString: true })

const object = yup.object().required()

const date = yup
  .date()
  .max(new Date(new Date().setDate(new Date().getDate() - 1)), 'No future date')
  .required()

const textNumericWithSpecialCharactersSchema = yup
  .string()
  .trim()
  .matches(textNumericWithSpecialCharactersPattern, Errors.notValid)
  .max(100)
  .required()

/* TODO:! rebuild with check equal password  and confirm password*/
export const checkPasswordSchema = yup
  .string()
  .matches(passwordPattern, Errors.passwordError)
  .min(6)
  .max(50)
  .label('Password')
  .required()

/* ------***------***---SIGN UP SCHEMS ---***------***------ */

export const signUpSchemaFirstStep = yup.object().shape({
  firstName: regName.label('First name'),
  lastName: regName.label('Last name'),
  phone: textNumericSchema,
  email: emailSchema
    // @ts-ignore
    .isEmailExist(),
})

export const signUpSchemaSecondStep = yup.object().shape({
  experience: numericSchema.required(),
  linkedIn: linkedInSchema.required(),
})

export const setPasswordSchema = yup.object().shape({
  password: checkPasswordSchema.label('Create password'),
  confirmPassword: yup
    .string()
    .oneOf([yup.ref('password'), null], 'Passwords must match')
    .label('Confirm password')
    .required(),
})

export const editProfileInformation = yup.object().shape({
  firstName: regName.label('First name'),
  lastName: regName.label('Last name'),
  phone: textNumericSchema,
  email: emailSchema.notRequired(),
})

export const updateProfileInfo = yup
  .mixed()
  .concat(editProfileInformation)
  .concat(signUpSchemaSecondStep)

export const updatePaymentDetails = yup.object().shape({
  abn: textNumericSchema
    .label('ABN')
    .notRequired()
    .nullable(),
  tfn: textNumericSchema
    .label('TFN')
    .notRequired()
    .nullable(),
  bsb: textNumericSchema
    .label('BSB')
    .notRequired()
    .nullable(),
  account: textNumericSchema
    .label('Account')
    .notRequired()
    .nullable(),
})

export const updatePassword = yup.object().shape({
  pass: textNumericSchema.label('Old password'),
  newPass: checkPasswordSchema.label('New password'),
  confirmNewPassword: yup
    .string()
    .oneOf([yup.ref('newPass'), null], 'Passwords must match')
    .label('Confirm password')
    .required(),
})

export const signUpSchemaFinishRegistration = yup.mixed().concat(signUpSchemaSecondStep)

/* -------------------------------------------------------------- */

/* ---***------***------***------***------***------***------***------***------***--- */

/* ------***------***---ADD EMPLOYER SCHEMS ---***------***------ */

export const addEmployerSchemaFirstStep = yup.object().shape({
  companyName: textNumericSchema.label('Company name'),
  City: yup
    .object()
    .label('City')
    .required(),
  address: textNumericWithSpecialCharactersSchema.label('HQ address'),
  abn: numericSchema
    .label('ABN')
    .required()
    .length(11)
    .min(11),
  phone: numericSchemaWithSpaces.nullable(),
})

export const addEmployerSchemaSecondStep = yup.object().shape({
  name: textSchema.label('Contact name'),
  position: textSchema
    .label('Contact position')
    .notRequired()
    .nullable(),
  email: emailSchema.label('Contact email'),
  phone: phoneSchema.nullable(),
})

/* ---***------***------***------***------***------***------***------***------***--- */

/* ------***------***---ADD JOB POSITION SCHEMS ---***------***------ */

export const addJobPositionSchemaFirstStep = yup.object().shape({
  Position: object.label('Job position title'),
  Sector: object.label('Project type'),
  City: object.label('City'),
  salary: numericSchema.label('Salary').required(),
})

export const addJobPositionSchemaSecondStep = yup.object().shape({
  name: textSchema.label('Hiring manager name'),
  position: textSchema.notRequired().nullable(),
  email: emailSchema.label('Hiring manager email address'),
  phone: phoneSchema.nullable(),
})

export const addJobPositionSchemaThirdStep = yup.object().shape({
  description: yup.string(),
})

/* ---***------***------***------***------***------***------***------***------***--- */

/* ------***------***---ADD CANDIDATE SCHEMS ---***------***------ */
export const addCandidateSchemaFirstStep = yup.object().shape({
  firstName: textNumericSchema.label('First name'),
  lastName: textNumericSchema.label('Last name'),
  birthday: date
    .notRequired()
    .max(new Date(new Date().getFullYear() - 10, new Date().getMonth(), new Date().getDate()))
    .min(new Date(new Date().getFullYear() - 100, new Date().getMonth(), new Date().getDate())),
  City: yup
    .object()
    .label('City')
    .required(),
  email: emailSchema
    .label('Email')
    .notRequired()
    .nullable(),
})

/* TODO:! Create concating flow of below types validateion (repeat the same things) */
export const workPlace = yup.object().shape({
  companyName: textNumericSchema.label('Company name'),
  companyWebsite: yup
    .string()
    .nullable()
    .label('Company webiste'),
  City: yup
    .object()
    .label('City')
    .required(),
  Position: yup
    .object()
    .label('Job position')
    .required(),
  current: yup.bool(),
  fromDate: date
    .label('From')
    .typeError('required field')
    .required(),
  toDate: date
    .label('To')
    .when('current', {
      is: false,
      then: date.min(yup.ref('fromDate'), 'not earlier than From'),
      otherwise: yup
        .date()
        .notRequired()
        .nullable(),
    })
    .notRequired()
    .nullable(),
})

export const addCandidateSecondStepWithoutWorkPlace = yup.object().shape({
  minSalary: numericSchema.label('Min salary').required(),
  availability: yup
    .date()
    .required()
    .typeError('not empty'),
  linkedIn: linkedInSchema,
})

export const addCandidateSchemaSecondStep = yup.object().shape({
  minSalary: numericSchema.label('Min salary').required(),
  availability: yup
    .date()
    .label('Availability')
    .required()
    .min(new Date(new Date().getFullYear(), new Date().getMonth(), new Date().getDate()))
    .max(new Date(new Date().getFullYear() + 10, new Date().getMonth(), new Date().getDate())),
  linkedIn: linkedInSchema,
  wpCurrent: yup.bool(),
  wpCompanyName: textNumericSchema.label('Company name'),
  wpCompanyWebsite: yup
    .string()
    .nullable()
    .label('Company webiste'),
  wpCity: yup
    .object()
    .label('City')
    .required(),
  wpPosition: yup
    .object()
    .label('Job position')
    .required(),
  wpFromDate: date
    .label('From')
    .typeError('required field')
    .min(new Date(new Date().getFullYear() - 100, new Date().getMonth(), new Date().getDate()))
    .required(),
  wpToDate: date
    .label('To')
    .when('wpCurrent', {
      is: false,
      then: date.min(yup.ref('wpFromDate'), 'not earlier than From'),
      otherwise: yup
        .date()
        .notRequired()
        .nullable(),
    })
    .notRequired()
    .nullable(),
})

/* ---***------***------***------***------***------***------***------***------***--- */

/* ------***------***---EDIT CANDIDATE SCHEMS ---***------***------ */

export const updateEditProfilePhoto = yup.object()

/* ---***------***------***------***------***------***------***------***------***--- */

/* ------***------***---  GOALS SCHEME ---***------***------ */

export const correctGoalsScheme = yup.object().shape({
  annualGoal: yup
    .number()
    .test('int', 'Must be integer', val => {
      return val % 1 === 0
    })
    .label('Annual Goal')
    .required(),
  monthlyGoal: yup
    .number()
    .test('int', 'Must be integer', val => {
      return val % 1 === 0
    })
    .label('Monthly Goal')
    .required(),
  quarterGoal: yup
    .number()
    .test('int', 'Must be integer', val => {
      return val % 1 === 0
    })
    .label('Quarter Goal')
    .required(),
  weeklyGoal: yup
    .number()
    .test('int', 'Must be integer', val => {
      return val % 1 === 0
    })
    .label('Weekly Goal')
    .required(),
})

/* ---***------***------***------***------***------***------***------***------***--- */

export const resetPasswordSchema = emailSchema

export const validateField = (name: string, value: string) => {
  switch (name) {
    case 'day':
      if (+value > 31 || +value === 0) return 'Invalid day'
      break
    case 'month':
      if (+value > 12 || +value === 0) return 'Invalid month'
      break
    case 'year':
      if (+value < new Date().getFullYear()) return
      if (+value > new Date().getFullYear() + 10) return `Invalid year maximum ${new Date().getFullYear() + 10}`
      break
    default:
      return ''
  }
  return ''
}
```
### actions employers
```js
import { createAction, ActionsUnion, IQueryParams } from '../../types/helps'
import {
  IDataEmployersLiveJobs,
  IDataEmployerWithPagedJobPositions,
  IDataJobPositionWithPagedPotentialCandidates,
} from '../../types/responseData'
import { IJobPosition, IEmployer, IEmployerContact } from '../../types/models'
import { IError, INotValidFieldsErr } from '../../types/errors'

export const FETCH_ALL_EMPLOYERS_STARTED = 'FETCH_ALL_EMPLOYERS_STARTED'
export const FETCH_ALL_EMPLOYERS_SUCCESS = 'FETCH_ALL_EMPLOYERS_SUCCESS'
export const FETCH_ALL_EMPLOYERS_FAIL = 'FETCH_ALL_EMPLOYERS_FAIL'

export const FETCH_EMPLOYER_STARTED = 'FETCH_EMPLOYER_STARTED'
export const FETCH_EMPLOYER_SUCCESS = 'FETCH_EMPLOYER_SUCCESS'
export const FETCH_EMPLOYER_FAIL = 'FETCH_EMPLOYER_FAIL'

export const CREATE_EMPLOYER_STARTED = 'CREATE_EMPLOYER_STARTED'
export const CREATE_EMPLOYER_SUCCESS = 'CREATE_EMPLOYER_SUCCESS'
export const CREATE_EMPLOYER_FAIL = 'CREATE_EMPLOYER_FAIL'

export const CREATE_EMPLOYER_CONTACT_STARTED = 'CREATE_EMPLOYER_CONTACT_STARTED'
export const CREATE_EMPLOYER_CONTACT_SUCCESS = 'CREATE_EMPLOYER_CONTACT_SUCCESS'
export const CREATE_EMPLOYER_CONTACT_FAIL = 'CREATE_EMPLOYER_CONTACT_FAIL'

export const UPDATE_EMPLOYER_CONTACT_STARTED = 'UPDATE_EMPLOYER_CONTACT_STARTED'

export const DELETE_EMPLOYER_CONTACT_STARTED = 'DELETE_EMPLOYER_CONTACT_STARTED'
export const DELETE_EMPLOYER_CONTACT_SUCCESS = 'DELETE_EMPLOYER_CONTACT_SUCCESS'
export const DELETE_EMPLOYER_CONTACT_FAIL = 'DELETE_EMPLOYER_CONTACT_FAIL'

export const CREATE_EMPLOYER_AND_EMPLOYER_CONTACT_STARTED = 'CREATE_EMPLOYER_AND_EMPLOYER_CONTACT_STARTED'

export const UPDATE_EMLOYER_STARTED = 'UPDATE_EMLOYER_STARTED'
export const UPDATE_EMLOYER_SUCCESS = 'UPDATE_EMLOYER_SUCCESS'
export const UPDATE_EMLOYER_FAIL = 'UPDATE_EMLOYER_FAIL'

export const DELETE_EMLOYER_STARTED = 'DELETE_EMLOYER_STARTED'
export const DELETE_EMLOYER_SUCCESS = 'DELETE_EMLOYER_SUCCESS'
export const DELETE_EMLOYER_FAIL = 'DELETE_EMLOYER_FAIL'

export const CREATE_JOB_STARTED = 'CREATE_JOB_STARTED'
export const CREATE_JOB_SUCCESS = 'CREATE_JOB_SUCCESS'
export const CREATE_JOB_FAIL = 'CREATE_JOB_FAIL'

export const FETCH_JOB_POSITION_STARTED = 'FETCH_JOB_POSITION_STARTED'
export const FETCH_NEXT_PAGE_JOB_POSITION_STARTED = 'FETCH_NEXT_PAGE_JOB_POSITION_STARTED'
export const FETCH_JOB_POSITION_SUCCESS = 'FETCH_JOB_POSITION_SUCCESS'
export const FETCH_JOB_POSITION_FAIL = 'FETCH_JOB_POSITION_FAIL'

export const SET_STATUS_JOB_POSITION = 'SET_STATUS_JOB_POSITION'
export const SET_STATUS_JOB_POSITION_SUCCESS = 'SET_STATUS_JOB_POSITION_SUCCESS'
export const SET_STATUS_JOB_POSITION_FAIL = 'SET_STATUS_JOB_POSITION_FAIL'

export const UPDATE_JOB_POSITION_STARTED = 'UPDATE_JOB_POSITION_STARTED'
export const UPDATE_JOB_POSITION_SUCCESS = 'UPDATE_JOB_POSITION_SUCCESS'
export const UPDATE_JOB_POSITION_FAIL = 'UPDATE_JOB_POSITION_FAIL'

export const DELETE_JOB_POSITION_STARTED = 'DELETE_JOB_POSITION_STARTED'
export const DELETE_JOB_POSITION_SUCCESS = 'DELETE_JOB_POSITION_SUCCESS'
export const DELETE_JOB_POSITION_FAIL = 'DELETE_JOB_POSITION_FAIL'

export const RESET_STORE = 'RESET_STORE'

export type EmployersTypes = ActionsUnion<typeof EmployersActions>

const EmployersActions = {
  fetchEmployers: (query: IQueryParams) => createAction(FETCH_ALL_EMPLOYERS_STARTED, query),
  fetchEmployersSuccess: (data: IDataEmployersLiveJobs) => createAction(FETCH_ALL_EMPLOYERS_SUCCESS, data),
  fetchEmployersFail: (error: IError) => createAction(FETCH_ALL_EMPLOYERS_FAIL, error),

  fetchEmployer: (employerId: number, page?: number, perPage?: number) =>
    createAction(FETCH_EMPLOYER_STARTED, { employerId, page, perPage }),
  fetchEmployerSuccess: (data: IDataEmployerWithPagedJobPositions) => createAction(FETCH_EMPLOYER_SUCCESS, data),
  fetchEmployerFail: (error: IError) => createAction(FETCH_EMPLOYER_FAIL, error),

  createEmployer: (employer: Partial<IEmployer>) => createAction(CREATE_EMPLOYER_STARTED, employer),
  createEmployerSuccess: (employer: IEmployer) => createAction(CREATE_EMPLOYER_SUCCESS, employer),
  createEmployerFail: (error: INotValidFieldsErr) => createAction(CREATE_EMPLOYER_FAIL, error),

  createEmployerContact: (employerId: number, contact: Partial<IEmployerContact>) =>
    createAction(CREATE_EMPLOYER_CONTACT_STARTED, { employerId, contact }),
  createEmployerContactSuccess: (contact: IEmployerContact) => createAction(CREATE_EMPLOYER_CONTACT_SUCCESS, contact),
  createEmployerContactFail: (error: INotValidFieldsErr) => createAction(CREATE_EMPLOYER_CONTACT_FAIL, error),

  updateEmployerContact: (contactId: number, contact: Partial<IEmployerContact>) =>
    createAction(UPDATE_EMPLOYER_CONTACT_STARTED, { contactId, contact }),

  deleteEmployerContact: (contactId: number) => createAction(DELETE_EMPLOYER_CONTACT_STARTED, contactId),
  deleteEmployerContactSuccess: (contactId: number) => createAction(DELETE_EMPLOYER_CONTACT_SUCCESS, contactId),
  deleteEmployerContactFail: (error: any) => createAction(DELETE_EMPLOYER_CONTACT_FAIL, error),

  createEmployerAndContact: (employer: Partial<IEmployer>, contact: Partial<IEmployerContact>) =>
    createAction(CREATE_EMPLOYER_AND_EMPLOYER_CONTACT_STARTED, { employer, contact }),

  updateEmployer: (id: number, employer: Partial<IEmployer>) => createAction(UPDATE_EMLOYER_STARTED, { id, employer }),
  updateEmployerSuccess: () => createAction(UPDATE_EMLOYER_SUCCESS),
  updateEmployerFail: (error: any) => createAction(UPDATE_EMLOYER_FAIL, error),

  deleteEmployer: (id: number) => createAction(DELETE_EMLOYER_STARTED, id),
  deleteEmployerSuccess: () => createAction(DELETE_EMLOYER_SUCCESS),
  deleteEmployerFail: (error: any) => createAction(DELETE_EMLOYER_FAIL, error),

  createJob: (employerId: number, job: Partial<IJobPosition>, contact?: undefined | Partial<IEmployerContact>) =>
    createAction(CREATE_JOB_STARTED, { employerId, job, contact }),
  createJobSuccess: (job: IJobPosition) => createAction(CREATE_JOB_SUCCESS, job),
  createJobFail: (error: INotValidFieldsErr) => createAction(CREATE_JOB_FAIL, error),

  fetchJobWithCandidates: (jobPositionId: number, page?: number, perPage?: number) =>
    createAction(FETCH_JOB_POSITION_STARTED, { jobPositionId, page, perPage }),
  fetchNextPageJobWithCandidates: (data: IDataJobPositionWithPagedPotentialCandidates) =>
    createAction(FETCH_NEXT_PAGE_JOB_POSITION_STARTED, data),
  fetchJobWithCandidatesSuccess: (data: IDataJobPositionWithPagedPotentialCandidates) =>
    createAction(FETCH_JOB_POSITION_SUCCESS, data),
  fetchJobWithCandidatesFail: (error: IError) => createAction(FETCH_JOB_POSITION_FAIL, error),

  updateJob: (id: number, job: Partial<IJobPosition>) => createAction(UPDATE_JOB_POSITION_STARTED, { id, job }),
  updateJobSuccess: (id: number, job: Partial<IJobPosition>) => createAction(UPDATE_JOB_POSITION_SUCCESS, { id, job }),
  updateJobFail: (error: any) => createAction(UPDATE_JOB_POSITION_FAIL, error),

  setStatusJobPosition: (id: number, value: boolean) => createAction(SET_STATUS_JOB_POSITION, { id, value }),
  setStatusJobPositionSuccess: (jobPosition: IJobPosition) =>
    createAction(SET_STATUS_JOB_POSITION_SUCCESS, jobPosition),
  setStatusJobPositionFail: (error: IError) => createAction(SET_STATUS_JOB_POSITION_FAIL, error),

  deleteJob: (id: number) => createAction(DELETE_JOB_POSITION_STARTED, id),
  deleteJobSuccess: (id: number) => createAction(DELETE_JOB_POSITION_SUCCESS, id),
  deleteJobFail: (error: any) => createAction(DELETE_JOB_POSITION_FAIL, error),

  resetStore: () => createAction(RESET_STORE),
}

export default EmployersActions
```
### Sagas employers
```js
import { call, put, takeLatest } from 'redux-saga/effects'
import EmployersActions, {
  FETCH_ALL_EMPLOYERS_STARTED,
  FETCH_EMPLOYER_STARTED,
  CREATE_EMPLOYER_STARTED,
  CREATE_EMPLOYER_CONTACT_STARTED,
  UPDATE_EMLOYER_STARTED,
  DELETE_EMLOYER_STARTED,
  CREATE_JOB_STARTED,
  FETCH_JOB_POSITION_STARTED,
  SET_STATUS_JOB_POSITION,
  UPDATE_JOB_POSITION_STARTED,
  DELETE_JOB_POSITION_STARTED,
  CREATE_EMPLOYER_AND_EMPLOYER_CONTACT_STARTED,
  DELETE_EMPLOYER_CONTACT_STARTED,
  UPDATE_EMPLOYER_CONTACT_STARTED,
} from './actions'
import EmployersApi from '../../api/employers'
import { formatErrorsForResponse } from '../../services/validation'

function* fetchEmplyersFlow(action: ReturnType<typeof EmployersActions.fetchEmployers>) {
  try {
    const { data } = yield call(EmployersApi.getLiveJobs, action.payload)
    yield put(EmployersActions.fetchEmployersSuccess(data))
  } catch (err) {
    yield put(EmployersActions.fetchEmployersFail(err))
  }
}

function* fetchEmployerFlow(action: ReturnType<typeof EmployersActions.fetchEmployer>) {
  try {
    const { employerId, page, perPage } = action.payload
    const { data } = yield call(EmployersApi.getEmployers, employerId, page, perPage)
    yield put(EmployersActions.fetchEmployerSuccess(data))
  } catch (err) {
    yield put(EmployersActions.fetchEmployerFail(err))
  }
}

function* createEmployerFlow(action: ReturnType<typeof EmployersActions.createEmployer>) {
  try {
    const form = Object.assign({}, action.payload, {
      CityId: action.payload.City!.id,
    })
    const { data } = yield call(EmployersApi.createEmployer, form)
    yield put(EmployersActions.createEmployerSuccess(data))
  } catch (err) {
    yield put(EmployersActions.createEmployerFail(err))
  }
}

function* updateEmployerFlow(action: ReturnType<typeof EmployersActions.updateEmployer>) {
  try {
    const { City, website, phone } = action.payload.employer
    const form = Object.assign({}, action.payload.employer, {
      CityId: City!.id,
      website: website || '',
      phone: phone || '',
    })
    yield call(EmployersApi.updateEmployer, action.payload.id, form)
    yield put(EmployersActions.updateEmployerSuccess())
  } catch (err) {
    yield put(EmployersActions.updateEmployerFail(err))
  }
}

function* deleteEmployerFlow(action: ReturnType<typeof EmployersActions.deleteEmployer>) {
  try {
    yield call(EmployersApi.deleteEmployer, action.payload)
    yield put(EmployersActions.deleteEmployerSuccess())
  } catch (err) {
    yield put(EmployersActions.deleteEmployerFail(err))
  }
}

function* createEmployerContactFlow(action: ReturnType<typeof EmployersActions.createEmployerContact>) {
  try {
    const form = Object.assign({}, action.payload.contact)
    const { data } = yield call(EmployersApi.createEmployerContact, action.payload.employerId, form)
    yield put(EmployersActions.createEmployerContactSuccess(data))
  } catch (err) {
    const { notValidFields } = err.response!.data!
    if (notValidFields) {
      const errors = formatErrorsForResponse(notValidFields)
      yield put(EmployersActions.createEmployerContactFail(errors))
    }
  }
}

function* updateEmployerContactFlow(action: ReturnType<typeof EmployersActions.updateEmployerContact>) {
  try {
    const form = Object.assign({}, action.payload.contact)
    yield call(EmployersApi.editEmployerContact, action.payload.contactId, form)
    // yield put(EmployersActions.createEmployerSuccess(data))
  } catch (err) {
    // yield put(EmployersActions.createEmployerFail(err))
  }
}

function* deleteEmployerContactFlow(action: ReturnType<typeof EmployersActions.deleteEmployerContact>) {
  try {
    yield call(EmployersApi.deleteEmployerContact, action.payload)
    yield put(EmployersActions.deleteEmployerContactSuccess(action.payload))
  } catch (err) {
    yield put(EmployersActions.deleteEmployerContactFail(err))
  }
}

function* createEmployerAndContactFlow(action: ReturnType<typeof EmployersActions.createEmployerAndContact>) {
  try {
    const formEmployer = Object.assign({}, action.payload.employer, {
      CityId: action.payload.employer.City!.id,
    })
    const { data } = yield call(EmployersApi.createEmployer, formEmployer)
    const formContact = Object.assign({}, action.payload.contact)
    yield call(EmployersApi.createEmployerContact, data.id, formContact)
    yield put(EmployersActions.createEmployerSuccess(data))
  } catch (err) {
    // TODO: need to fix the error handling that the server sends
    yield put(EmployersActions.createEmployerFail(err))
  }
}

function* createJobPositionFlow(action: ReturnType<typeof EmployersActions.createJob>) {
  try {
    const form = Object.assign({}, action.payload.job, {
      CityId: action.payload.job.City!.id,
      StateId: action.payload.job.City!.State.id,
      PositionId: action.payload.job.Position!.id,
      SectorId: action.payload.job.Sector!.id,
      salary: Number(action.payload.job.salary),
    })
    if (action.payload.contact) {
      const { data: contact } = yield call(
        EmployersApi.createEmployerContact,
        action.payload.employerId,
        action.payload.contact
      )
      form.EmployerContactId = contact.id
    }
    const { data } = yield call(EmployersApi.createJobPosition, action.payload.employerId, form)
    yield put(EmployersActions.createJobSuccess(data))
  } catch (err) {
    yield put(EmployersActions.createJobFail(err))
  }
}

function* fetchJobPositionFlow(action: ReturnType<typeof EmployersActions.fetchJobWithCandidates>) {
  try {
    const { jobPositionId, page, perPage } = action.payload
    const { data } = yield call(EmployersApi.getJobPositionWithPotentialCandidates, jobPositionId, page, perPage)
    if (page) {
      yield put(EmployersActions.fetchNextPageJobWithCandidates(data))
    } else {
      yield put(EmployersActions.fetchJobWithCandidatesSuccess(data))
    }
  } catch (err) {
    yield put(EmployersActions.fetchJobWithCandidatesFail(err))
  }
}

function* updateJobPositionFlow(action: ReturnType<typeof EmployersActions.updateJob>) {
  try {
    const form = {
      ...action.payload.job,
      salary: Number(action.payload.job.salary),
      closed: Boolean(action.payload.job.closed),
      PositionId: Number(action.payload.job.Position!.id),
      SectorId: Number(action.payload.job.Sector!.id),
      CityId: Number(action.payload.job.City!.id),
      // managerPhone: action.payload.job.managerPhone || '',
      // managerPosition: action.payload.job.managerPosition || '',
      description: action.payload.job.description || '',
    }
    const { data } = yield call(EmployersApi.updateJobPosition, action.payload.id, form)
    yield put(EmployersActions.updateJobSuccess(action.payload.id, data))
  } catch (err) {
    yield put(EmployersActions.updateJobFail(err))
  }
}

function* setStatusJobPositionFlow(action: ReturnType<typeof EmployersActions.setStatusJobPosition>) {
  try {
    const { data } = yield call(EmployersApi.setStatusJobPosition, action.payload.id, action.payload.value)
    yield put(EmployersActions.setStatusJobPositionSuccess(data))
  } catch (err) {
    yield put(EmployersActions.setStatusJobPositionFail(err))
  }
}

function* deleteJobPositionFlow(action: ReturnType<typeof EmployersActions.deleteJob>) {
  try {
    yield call(EmployersApi.deleteJobPosition, action.payload)
    yield put(EmployersActions.deleteJobSuccess(action.payload))
  } catch (err) {
    yield put(EmployersActions.updateJobFail(err))
  }
}

export default function* watchEmployers() {
  yield takeLatest(FETCH_ALL_EMPLOYERS_STARTED, fetchEmplyersFlow)
  yield takeLatest(FETCH_EMPLOYER_STARTED, fetchEmployerFlow)
  yield takeLatest(CREATE_EMPLOYER_STARTED, createEmployerFlow)
  yield takeLatest(CREATE_EMPLOYER_CONTACT_STARTED, createEmployerContactFlow)
  yield takeLatest(UPDATE_EMPLOYER_CONTACT_STARTED, updateEmployerContactFlow)
  yield takeLatest(CREATE_EMPLOYER_AND_EMPLOYER_CONTACT_STARTED, createEmployerAndContactFlow)
  yield takeLatest(UPDATE_EMLOYER_STARTED, updateEmployerFlow)
  yield takeLatest(DELETE_EMLOYER_STARTED, deleteEmployerFlow)
  yield takeLatest(CREATE_JOB_STARTED, createJobPositionFlow)
  yield takeLatest(FETCH_JOB_POSITION_STARTED, fetchJobPositionFlow)
  yield takeLatest(SET_STATUS_JOB_POSITION, setStatusJobPositionFlow)
  yield takeLatest(UPDATE_JOB_POSITION_STARTED, updateJobPositionFlow)
  yield takeLatest(DELETE_JOB_POSITION_STARTED, deleteJobPositionFlow)
  yield takeLatest(DELETE_EMPLOYER_CONTACT_STARTED, deleteEmployerContactFlow)
}
```
### Store employers
```js
import { fromJS, List } from 'immutable'
import {
  CREATE_EMPLOYER_FAIL,
  EmployersTypes,
  FETCH_ALL_EMPLOYERS_FAIL,
  FETCH_ALL_EMPLOYERS_STARTED,
  FETCH_ALL_EMPLOYERS_SUCCESS,
} from './actions'
import { IEmployersState } from '../../types/reducerStates'

const initialState: IEmployersState = fromJS({
  isFetching: false,
  query: {
    page: null,
    perPage: null,
  },
  items: List([]),
  employersQty: null,
  pagesQty: null,
  error: null,
})

export default (state = initialState, action: EmployersTypes): IEmployersState => {
  switch (action.type) {
    case FETCH_ALL_EMPLOYERS_STARTED:
      return state.set('isFetching', true)
    case FETCH_ALL_EMPLOYERS_SUCCESS:
      return state
        .set('isFetching', false)
        .set('items', List(action.payload.employers))
        .set('query', action.payload.query)
        .set('employersQty', action.payload.employersQty)
        .set('pagesQty', action.payload.pagesQty)
        .set('error', null)
    case FETCH_ALL_EMPLOYERS_FAIL:
      return state.set('isFetching', false).set('error', action.payload)
    case CREATE_EMPLOYER_FAIL:
      return state.set('error', action.payload)
    default:
      return state
  }
}
```
### types for create actions
```js
import React from 'react'
import { ValidationError } from 'yup'

export type InputEvent = React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement>
export type ValidationError = ValidationError

export interface IPaginationBasic {
  query: {
    page: number
    perPage: number
  } | null
  pagesQty: null | number
}

export interface IQueryParams {
  page?: number | string
  perPage?: number | string
  sort?: 'recent-activity' | 'name' | 'job-positions'
  order?: 'asc' | 'desc'
}

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
### Card
```js
import React from 'react'
import moment from 'moment'
import Avatar from '../Avatar'
import { TypeItem } from '../../containers/Candidates/Candidate/styles'
import { IJobPositionsWithMathcedCandidates } from '../../types/models'
import { BoxFill, Box, BoxCol, Text, Bold, BoxCenterContent } from '../../styles'
import { formatCurrency } from '../../utils'

type Props = {
  job: IJobPositionsWithMathcedCandidates
  match: (JobPositionId: number) => void
  onClick: (job: IJobPositionsWithMathcedCandidates) => void
}

const PotentialJobCard: React.FC<Props> = ({ job, match, onClick }) => {
  const abbrJob = job.Employer.companyName.split(' ').map(char => char[0])
  const setJobId = () => onClick(job)
  const matchCandidate = (e: React.MouseEvent) => {
    e.stopPropagation()
    match(job.id)
  }
  return (
    <BoxFill allPointer defaultBorder smooth column shadow2 whiteBC mb="10px" padding="20px" onClick={setJobId}>
      <BoxFill spaceBetween mb="16px">
        <Box alignCenter>
          <Box>
            <Box zIndex="5">
              <Avatar chars={abbrJob} width="49px" />
            </Box>
            <BoxCenterContent whiteBC width="49px" height="49px" ml="-10px" zIndex="5" br="50%" shadow2>
              <Text xs black>
                {job.complianceDegree}%
              </Text>
            </BoxCenterContent>
          </Box>
          <BoxCol ml="10px">
            <Bold l black left>
              {job.Position.name}
            </Bold>
            <Box>
              <Box pr="8px" borderRight="1px solid #E3E8F1">
                <Text black>{job.Employer.companyName}</Text>
              </Box>
              <Box ml="8px">
                <Text>
                  {job.City.name}
                  {job.City.State.name}
                </Text>
              </Box>
            </Box>
          </BoxCol>
        </Box>
        <Box alignCenter>
          <TypeItem isSector text={job.Sector.name} fz="14px" padding="10px 14px" />
          <Box ml="7px" smooth bc="#D8F0E9" padding="10px 14px">
            <Text s secondColor>
              ${formatCurrency(job.salary)}
            </Text>
          </Box>
        </Box>
      </BoxFill>
      {!job.description || (
        <BoxFill maxHeight="32px" overflow="hidden" mb="16px">
          <Text lh="16px" black s left>
            {job.description}
          </Text>
        </BoxFill>
      )}
      <BoxFill alignCenter spaceBetween>
        <Box alignCenter>
          <BoxCenterContent onClick={matchCandidate} smooth bc="#D8F0E9" padding="12px 40px">
            <Text secondColor>MATCH</Text>
          </BoxCenterContent>
          <Box ml="16px">
            <Text s>Added {moment(job.createdAt).fromNow()}</Text>
          </Box>
        </Box>
        <Box>
          {job.Candidates.map(candidate => (
            <Box key={candidate.id} ml="-10px">
              <Avatar item={candidate} width="38px" />
            </Box>
          ))}
        </Box>
      </BoxFill>
    </BoxFill>
  )
}

export default PotentialJobCard
```
