<template>
  <div>
    <v-alert type="warning" v-model="alert" dismissible>
      This user already exists, please check the account carefully!
    </v-alert>
    <v-data-table
      :headers="headers"
      :items="users"
      :items-per-page="15"
      class="elevation-1"
      :loading="userLoading"
      loading-text="Loading... Please wait"
      :search="search"
      :footer-props="{
        showFirstLastPage: true,
        firstIcon: 'mdi-arrow-collapse-left',
        lastIcon: 'mdi-arrow-collapse-right',
        prevIcon: 'mdi-arrow-left',
        nextIcon: 'mdi-arrow-right',
      }"
    >
      <template v-slot:top>
        <v-toolbar flat>
          <v-toolbar-title>User List</v-toolbar-title>
          <v-spacer></v-spacer>
          <v-text-field
            v-model="search"
            append-icon="mdi-magnify"
            label="Search"
            single-line
            hide-details
          ></v-text-field>
          <v-spacer></v-spacer>
          <v-dialog v-model="dialog" max-width="500px">
            <template v-slot:activator="{ on, attrs }">
              <v-btn
                dark
                color="indigo"
                class="mx-2 mb-2"
                @click="downloadBtnClick"
                ><span class="pr-2" v-show="$vuetify.breakpoint.name !== 'xs'"
                  >Download</span
                >
                <v-icon dark> mdi-cloud-download </v-icon>
              </v-btn>
              <v-btn color="cyan" dark class="mb-2" v-bind="attrs" v-on="on">
                <span class="pr-2" v-show="$vuetify.breakpoint.name !== 'xs'"
                  >New User</span
                >
                <v-icon dark> mdi-account-plus </v-icon>
              </v-btn>
            </template>
            <v-card>
              <v-card-title>
                <span class="text-h5">Add New User</span>
              </v-card-title>
              <v-form ref="form" v-model="valid" lazy-validation>
                <v-card-text>
                  <v-text-field
                    v-model="newUser.email"
                    label="Email"
                    :rules="emailRules"
                    required
                  ></v-text-field>
                  <v-select
                    class="mt-4"
                    :items="roles"
                    v-model="newUser.role"
                    label="Role"
                    dense
                  ></v-select>
                  <v-icon>info</v-icon>
                  Read-Write: Represent normal user
                </v-card-text>
                <v-card-actions>
                  <v-spacer></v-spacer>
                  <v-btn color="blue darken-1" text @click="close"
                    >Cancel</v-btn
                  >
                  <v-btn
                    color="blue darken-1"
                    text
                    @click="save"
                    :disabled="!valid"
                    >Save</v-btn
                  >
                </v-card-actions>
              </v-form>
            </v-card>
          </v-dialog>
          <v-dialog v-model="dialogDelete" max-width="500px">
            <v-card>
              <v-card-title class="text-h5 grey lighten-2">
                <v-icon large color="red darken-2">mdi-close-circle</v-icon>
                Dangerous Operation</v-card-title
              >
              <v-card-title class="text-h5"
                >Are you sure to delete this user?</v-card-title
              >
              <v-card-actions>
                <v-spacer></v-spacer>
                <v-btn color="blue darken-1" text @click="closeDelete"
                  >Cancel</v-btn
                >
                <v-btn color="blue darken-1" text @click="deleteItemConfirm"
                  >OK</v-btn
                >
                <v-spacer></v-spacer>
              </v-card-actions>
            </v-card>
          </v-dialog>
        </v-toolbar>
      </template>
      <!-- eslint-disable-next-line -->
      <template v-slot:item.actions="{ item }">
        <v-btn color="primary" text @click="deleteItem(item)">
          <v-icon left> mdi-delete </v-icon>
          Delete
        </v-btn>
      </template>
    </v-data-table>
  </div>
</template>

<script>
import moment from 'moment'
import XLSX from 'xlsx'

export default {
  name: 'UserTable',
  data() {
    return {
      alert: false,
      search: '',
      userLoading: false,
      headers: [
        {
          text: 'Email',
          align: 'start',
          value: 'email',
          width: '45%'
        },
        {
          text: 'Role',
          align: 'start',
          value: 'role',
          width: '15%'
        },
        {
          text: 'Last Login Time',
          align: 'end',
          value: 'last_login_time',
          width: '25%'
        },
        {
          text: 'Actions',
          align: 'center',
          value: 'actions',
          sortable: false,
          width: '15%'
        }
      ],
      users: [],
      dialog: false,
      dialogDelete: false,
      editedIndex: -1,
      newUser: {
        email: '',
        role: 'rw'
      },
      valid: false,
      emptyUser: {
        email: '',
        role: 'rw'
      },
      emailRules: [
        v => !!v || 'E-mail is required',
        v => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v) || 'E-mail must be valid'
      ],
      roles: [
        {
          text: 'Admin',
          value: 'admin'
        },
        {
          text: 'Read-Write',
          value: 'rw'
        }
      ]
    }
  },
  created() {
    this.refreshUserList()
  },
  methods: {
    async refreshUserList() {
      this.userLoading = true
      const { data } = await this.$http.get('/user')
      this.users = data.users.map(e => {
        if (Object.prototype.hasOwnProperty.call(e, 'last_login_time')) {
          e.last_login_time = moment(e.last_login_time).format('YYYY-MM-DD HH:mm:ss')
        } else {
          e.last_login_time = 'Never'
        }
        e.role = e.role === 'admin' ? 'Admin' : 'Read-Write'
        return e
      })
      this.userLoading = false
    },
    close() {
      this.dialog = false
      this.$nextTick(() => {
        this.newUser = Object.assign({}, this.emptyUser)
      })
    },
    async save() {
      if (this.$refs.form.validate()) {
        // check for duplicate
        const lowerCaseEmail = this.newUser.email.toLowerCase()
        if (this.users.some(d => d.email.toLowerCase() === lowerCaseEmail)) {
          this.alert = true
        } else {
          // save new user to mongo db
          await this.$http.post('/user', this.newUser)
          // update the ui
          this.refreshUserList()
        }
        this.close()
      }
    },
    deleteItem(item) {
      this.editedIndex = this.users.indexOf(item)
      this.editedItem = Object.assign({}, item)
      this.dialogDelete = true
    },
    async deleteItemConfirm() {
      // send request to delete in db
      await this.$http.delete(`/user/${this.editedItem.email}`)
      this.users.splice(this.editedIndex, 1)
      this.closeDelete()
    },
    closeDelete() {
      this.dialogDelete = false
      this.$nextTick(() => {
        this.editedItem = Object.assign({}, this.emptyUser)
        this.editedIndex = -1
      })
    },
    downloadBtnClick() {
      const header = ['email', 'role', 'last_login_time']
      const usersSheet = XLSX.utils.json_to_sheet(this.users, { header })
      usersSheet.A1.v = 'Email'
      usersSheet.B1.v = 'Role'
      usersSheet.C1.v = 'Last Login Time'

      const wb = XLSX.utils.book_new()
      XLSX.utils.book_append_sheet(wb, usersSheet, 'users')
      const timeTag = moment().format('YYYYMMDDHHmmss')
      XLSX.writeFile(wb, `SL-${timeTag}-users.xlsx`)
    }
  },
  watch: {
    dialog(val) {
      val || this.close()
    },
    dialogDelete(val) {
      val || this.closeDelete()
    }
  }
}
</script>
