<template>
  <div class="d-inline-flex align-items-center">
    <label v-if="label" :for="`user-dropdown-${roleId}`" class="mb-0 mr-1">{{ label }}</label>
    <b-dropdown
        :id="`user-dropdown-${roleId}`"
        data-testid="user-dropdown"
        size="sm"
        class="users-dropdown"
        v-b-tooltip.topright="selectedTooltip ? getUsersStr() : ''">

      <!--   Text on dropdown (configurable via default slot)   -->
      <template #button-content>
        <slot :count="localSelectedUsers.length" :filter="searchTerm" :unknown="loading || error">
          {{ getUsersStr() }}
        </slot>
      </template><!--   /Text on dropdown (configurable via default slot)   -->

      <b-dropdown-form>
        <!--    search bar    -->
        <div class="w-100 sticky-top dropdown-sticky-search-bar-container">
          <b-input-group size="sm" class="hide-behind-search dropdown-sticky-search-bar">
            <template #prepend>
              <b-input-group-text>
                <span class="fa fa-search fa-fw" />
              </b-input-group-text>
            </template>
            <b-form-input
                autofocus
                type="text"
                debounce="400"
                v-model="searchTerm"
                placeholder="Begin typing to search for users by name or id"
            />
            <template #append>
              <b-button
                  :disabled="!searchTerm"
                  @click="searchTerm = ''"
                  variant="outline-secondary"
                  v-b-tooltip.hover="'Clear search'">
                <span class="fa fa-close" />
              </b-button>
            </template>
          </b-input-group>
        </div><!--    /search bar    -->

        <div class="w-100 dropdown-checkbox-pad">
          <!-- loading -->
          <template v-if="loading">
            <div class="mt-3 text-center">
              <span class="fa fa-circle-o-notch fa-spin fa-2x" />
              <p>Loading users...</p>
            </div>
          </template> <!-- /loading -->

          <!--    error      -->
          <template v-else-if="error">
            <div class="mt-3 alert alert-warning">
              <span class="fa fa-exclamation-triangle" />&nbsp;
              {{ error }}
            </div>
          </template><!--    /error      -->

          <!--     user checkboxes     -->
          <b-form-checkbox-group v-else class="d-flex flex-column"
                                 v-model="localSelectedUsers">
            <b-form-checkbox
                :key="user.userId"
                :value="user.userId"
                v-for="user in users"
                @change="updateUsers(user.userId, $event)">
              {{ user.userName }} ({{ user.userId }})
            </b-form-checkbox>
          </b-form-checkbox-group><!--     /user checkboxes     -->
        </div>
      </b-dropdown-form>
    </b-dropdown>
  </div>

</template>

<script>
import UserService from './UserService';

export default {
  name: 'UserDropdown',
  props: {
    roleId: {
      type: String,
      required: false // during creation, a role will not have an ID
    },
    selectedUsers: {
      type: Array,
      required: false // can use initializeSelectionWithRole instead
    },
    requestRoleStatus: { type: Boolean },
    initializeSelectionWithRole: { type: Boolean },
    selectedTooltip: { type: Boolean },
    label: {
      type: String,
      required: false
    }
  },
  data () {
    return {
      searchTerm: '',
      users: undefined,
      loading: true,
      error: '',
      localSelectedUsers: this.selectedUsers || []
    };
  },
  watch: {
    searchTerm () {
      this.loadUsers();
    }
  },
  methods: {
    getUsersStr () {
      const userArr = [...this.localSelectedUsers];
      userArr.sort();
      return userArr.join(', ');
    },
    loadUsers () {
      const query = {
        filter: this.searchTerm
      };
      if (this.requestRoleStatus && this.roleId != null) {
        query.roleId = this.roleId;
      }

      UserService.searchUsersMin(query).then((response) => {
        this.error = '';
        this.loading = false;
        this.users = JSON.parse(JSON.stringify(response.data));
        if (this.initializeSelectionWithRole) {
          this.localSelectedUsers = this.users.filter(u => u.hasRole).map(u => u.userId);
        }
      }).catch((error) => {
        this.loading = false;
        this.error = error.text;
      });
    },
    updateUsers (userId, newSelection) { // emits both the new array and changed user-value
      const change = {
        newSelection,
        changedUser: {
          userId,
          newState: newSelection.length > this.localSelectedUsers.length
        }
      };
      this.$emit('selected-users-updated', change, this.roleId);
    }
  },
  mounted () {
    this.loadUsers();
  }
};
</script>

<style scoped>
/* hides elements scrolling behind sticky search bar */
.hide-behind-search {
  padding-top: 0.5rem !important;
  background-color: var(--color-background) !important;
}

/* allows different apps to customize compactness of their user dropdowns */
.dropdown-sticky-search-bar-container, .dropdown-checkbox-pad {
  padding-inline: 1.5rem;
}
</style>
