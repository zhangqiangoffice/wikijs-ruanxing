<template lang="pug">
  v-dialog(v-model='isShown', max-width='650', persistent)
    v-card
      .dialog-header.is-short
        v-icon.mr-3(color='white') mdi-plus
        span Synchronize User
        v-spacer
        v-btn.mx-0(color='white', outlined, disabled, dark)
          v-icon(left) mdi-database-import
          span Bulk Import
      v-card-text.pt-5
        v-select(
          :items='providers'
          item-text='displayName'
          item-value='key'
          outlined
          prepend-icon='mdi-domain'
          v-model='provider'
          label='Provider'
          )
        v-text-field(
          outlined
          prepend-icon='mdi-account-outline'
          v-model='name'
          label='Name'
          :hint='provider === `local` ? `Users cannot change.` : `May be overwritten by the provider during login.`'
          key='newUserName'
          persistent-hint
          ref='nameInput'
          )
        v-select.mt-2(
          :items='groups'
          item-text='name'
          item-value='id'
          item-disabled='isSystem'
          outlined
          prepend-icon='mdi-account-group'
          v-model='group'
          label='Assign to Group(s)...'
          hint='Note that you cannot assign users to the Administrators or Guests groups from this dialog.'
          persistent-hint
          clearable
          multiple
          )
        v-divider
      v-card-chin
        v-spacer
        v-btn(text, @click='isShown = false') Cancel
        v-btn.px-3(depressed, color='primary', @click='newUser(false)')
          v-icon(left) mdi-chevron-right
          span Create
        v-btn.px-3(depressed, color='primary', @click='newUser(true)')
          v-icon(left) mdi-chevron-double-right
          span Create and Close
</template>

<script>
import _ from 'lodash'
import gql from 'graphql-tag'

import createUserMutation from 'gql/admin/users/users-mutation-create.gql'
import groupsQuery from 'gql/admin/users/users-query-groups.gql'

export default {
  props: {
    value: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      providers: [],
      provider: 'local',
      name: '',
      groups: [],
      group: [],
    }
  },
  computed: {
    isShown: {
      get() { return this.value },
      set(val) { this.$emit('input', val) }
    }
  },
  methods: {
    async newUser(close = false) {
      if (this.name === null || this.name === '') {
        this.$store.commit('showNotification', {
          style: 'red',
          message: 'The name is empty!',
          icon: 'alert'
        })
        return
      }

      try {
        const resp = await this.$apollo.mutate({
          mutation: createUserMutation,
          variables: {
            providerKey: this.provider,
            name: this.name,
            groups: this.group,
            email: '',
            passwordRaw: '',
            mustChangePassword: false,
            sendWelcomeEmail: false
          },
          watchLoading (isLoading) {
            this.$store.commit(`loading${isLoading ? 'Start' : 'Stop'}`, 'admin-users-create')
          }
        })
        if (_.get(resp, 'data.users.create.responseResult.succeeded', false)) {
          this.$store.commit('showNotification', {
            style: 'success',
            message: 'New user created successfully.',
            icon: 'check'
          })

          this.name = ''

          if (close) {
            this.isShown = false
            this.$emit('refresh')
          } else {
            this.$refs.nameInput.focus()
          }
        } else {
          this.$store.commit('showNotification', {
            style: 'red',
            message: _.get(resp, 'data.users.create.responseResult.message', 'An unexpected error occurred.'),
            icon: 'alert'
          })
        }
      } catch (err) {
        this.$store.commit('pushGraphError', err)
      }
    }
  },
  apollo: {
    providers: {
      query: gql`
        query {
          authentication {
            activeStrategies {
              key
              displayName
            }
          }
        }
      `,
      fetchPolicy: 'network-only',
      update: (data) => data.authentication.activeStrategies,
      watchLoading (isLoading) {
        this.$store.commit(`loading${isLoading ? 'Start' : 'Stop'}`, 'admin-users-strategies-refresh')
      }
    },
    groups: {
      query: groupsQuery,
      fetchPolicy: 'network-only',
      update: (data) => data.groups.list,
      watchLoading (isLoading) {
        this.$store.commit(`loading${isLoading ? 'Start' : 'Stop'}`, 'admin-auth-groups-refresh')
      }
    }
  }
}
</script>
