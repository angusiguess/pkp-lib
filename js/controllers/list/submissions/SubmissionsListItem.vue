<template>
	<li class="pkpListPanelItem pkpListPanelItem--submission" :class="{'--hasFocus': isFocused}">
		<a :href="item.urlWorkflow" class="pkpListPanelItem--submission__link" @focus="focusItem" @blur="blurItem">
			<div class="pkpListPanelItem--submission__item">
				<div class="pkpListPanelItem--submission__id">
					<span class="pkp_screen_reader">{{ i18n.id }}</span>
					{{ item.id }}
				</div>
				<div v-if="item.author" class="pkpListPanelItem--submission__author">
					{{ item.author.authorString }}
				</div>
				<div class="pkpListPanelItem--submission__title">
					{{ item.title }}
				</div>
				<div v-if="notice" class="pkpListPanelItem--submission__activity">
					<span class="fa fa-exclamation-triangle"></span>
					{{ notice }}
				</div>
			</div>
		</a>
		<div v-if="currentUserIsReviewer" class="pkpListPanelItem--submission__stage pkpListPanelItem--submission__stage--reviewer">
			<a :href="item.urlWorkflow" tabindex="-1">
				<div v-if="currentUserLatestReviewAssignment.responsePending" class="pkpListPanelItem--submission__dueDate">
					<div :aria-labelledby="responseDueLabelId" class="pkpListPanelItem--submission__dueDateValue">
						{{ currentUserLatestReviewAssignment.responseDue }}
					</div>
					<div :id="responseDueLabelId" class="pkpListPanelItem--submission__dueDateLabel">
						{{ i18n.responseDue }}
					</div>
				</div>
				<div v-if="currentUserLatestReviewAssignment.reviewPending" class="pkpListPanelItem--submission__dueDate">
					<div :aria-labelledby="reviewDueLabelId" class="pkpListPanelItem--submission__dueDateValue">
						{{ currentUserLatestReviewAssignment.due }}
					</div>
					<div :id="reviewDueLabelId" class="pkpListPanelItem--submission__dueDateLabel">
						{{ i18n.reviewDue }}
					</div>
				</div>
			</a>
		</div>
		<div v-else class="pkpListPanelItem--submission__stage">
			<div class="pkpListPanelItem--submission__stageRow">
				<div class="pkpListPanelItem--submission__stageLabel">
					<template v-if="item.status.id === 3 || item.status.id === 4">
						{{ item.status.label }}
					</template>
					<template v-else-if="item.submissionProgress > 0">
						{{ i18n.incomplete }}
					</template>
					<template v-else>
						{{ activeStage.label }}
					</template>
				</div>
				<div class="pkpListPanelItem--submission__flags">
					<span v-if="isReviewStage"  class="pkpListPanelItem--submission__flags--reviews">
						<span :aria-labelledby="reviewsCompletedLabelId" class="count">{{ completedReviewsCount }} / {{ currentReviewAssignments.length }}</span>
						<span :id="reviewsCompletedLabelId" class="pkp_screen_reader">{{ i18n.reviewsCompleted }}</span>
					</span>
					<span v-if="activeStage.files.count" class="pkpListPanelItem--submission__flags--files">
						<span :aria-labelledby="filesPreparedLabelId" class="count">{{ activeStage.files.count }}</span>
						<span :id="filesPreparedLabelId" class="pkp_screen_reader">{{ i18n.filesPrepared }}</span>
					</span>
					<span v-if="openQueryCount" class="pkpListPanelItem--submission__flags--discussions">
						<span :aria-labelledby="discussionsLabelId" class="count">{{ openQueryCount }}</span>
						<span :id="discussionsLabelId" class="pkp_screen_reader">{{ i18n.discussions }}</span>
					</span>
				</div>
			</div>
			<div v-if="hasActions" class="pkpListPanelItem__actions">
				<a v-if="currentUserCanDelete" href="#" class="delete" @click.prevent="deleteSubmissionPrompt" @focus="focusItem" @blur="blurItem">
					{{ i18n.delete }}
				</a>
				<a v-if="currentUserCanViewInfoCenter" class="pkpListPanelItem__openInfoCenter" href="#" @click.prevent="openInfoCenter" @focus="focusItem" @blur="blurItem">
					{{ i18n.infoCenter }}
				</a>
			</div>
		</div>
		<div class="pkpListPanelItem__mask" :class="classMask">
			<div class="pkpListPanelItem__maskLabel">
				<template v-if="mask === 'confirmingDelete'">
					<span class="pkpListPanelItem__maskLabel_prompt">
						{{ i18n.confirmDelete }}
						<a href="#" @click.prevent="deleteSubmission">Yes</a>
						<a href="#" @click.prevent="cancelDeleteRequest">No</a>
					</span>
				</template>
				<template v-if="mask === 'deleting'">
					<span class="pkpListPanelItem__maskLabel_loading">
						<span class="pkp_spinner"></span>
						{{ i18n.deleting }}
					</span>
				</template>
			</div>
		</div>
	</li>
</template>

<script>
import ListPanelItem from '../ListPanelItem.vue';

export default {
	extends: ListPanelItem,
	name: 'SubmissionsListItem',
	props: ['item', 'i18n', 'apiPath', 'infoUrl'],
	data: function() {
		return {
			mask: null,
		};
	},
	computed: {
		/**
		 * Map the submission id to the list item id
		 */
		id: function() {
			return this.item.id;
		},

		/**
		 * Can the current user delete a submission?
		 *
		 * @return bool
		 */
		currentUserCanDelete: function() {
			if (pkp.userHasRole(['manager', 'siteAdmin'])) {
				return true;
			} else if (pkp.userHasRole('author') && this.item.submissionProgress !== 0) {
				return true;
			}
			return false; // @todo
		},

		/**
		 * Can the current user view the info center?
		 *
		 * @return bool
		 */
		currentUserCanViewInfoCenter: function() {
			return pkp.userHasRole(['manager', 'subeditor', 'assistant']);
		},

		/**
		 * Is the current user a reviewer on this submission?
		 *
		 * @return bool
		 */
		currentUserIsReviewer: function() {
			var isReviewer = false;
			_.each(this.item.reviewAssignments, function(review) {
				if (review.isCurrentUserAssigned) {
					isReviewer = true;
					return;
				}
			});

			return isReviewer;
		},

		/**
		 * Are there any actions available for this submission?
		 *
		 * @return bool
		 */
		hasActions: function() {
			return this.currentUserCanDelete || this.currentUserCanViewInfoCenter;
		},

		/**
		 * The current stage
		 *
		 * @return array
		 */
		activeStage: function() {
			return _.findWhere(this.item.stages, {isActiveStage: true});
		},

		/**
		 * Compile a notice depending on the stage status
		 *
		 * Only stage status' that have pending work for the current user should
		 * result in a notice.
		 *
		 * @todo Set different notice priorities for different users. Current
		 *  set up is for editors.
		 * @return string
		 */
		notice: function() {
			var notice = '';

			// Notices for journal managers
			if (pkp.userHasRole('manager')) {
				if (this.activeStage.id === 1) {
					switch (this.activeStage.statusId) {
						case 1: // @todo this should be a global
							// Only display unassigned notice for completed submissions
							if (this.item.submissionProgress === 0) {
								notice = this.activeStage.status;
							}
							break;
					}
				}
			}

			// Notices for journal managers and subeditors
			if (pkp.userHasRole(['manager', 'subeditor'])) {
				if (this.isReviewStage) {
					switch (this.activeStage.statusId) {
						case 6: // REVIEW_ROUND_STATUS_PENDING_REVIEWERS
						case 8: // REVIEW_ROUND_STATUS_REVIEWS_READY
						case 9: // REVIEW_ROUND_STATUS_REVIEWS_COMPLETED
						case 10: // REVIEW_ROUND_STATUS_REVIEWS_OVERDUE
						case 11: // REVIEW_ROUND_STATUS_REVISIONS_SUBMITTED
							notice = this.activeStage.status;
							break;
					}
				}
			}

			// Notices for authors
			if (pkp.userHasRole(['author'])) {
				if (this.isReviewStage) {
					switch (this.activeStage.statusId) {
						case 1: // REVIEW_ROUND_STATUS_REVISIONS_REQUESTED
							notice = this.activeStage.status;
							break;
					}
				}
			}

			// Notices for reviewers
			if (this.currentUserIsReviewer) {
				switch (this.currentUserLatestReviewAssignment.statusId) {
					case 0: // REVIEW_ASSIGNMENT_STATUS_AWAITING_RESPONSE
					case 4: // REVIEW_ASSIGNMENT_STATUS_RESPONSE_OVERDUE
					case 6: // REVIEW_ASSIGNMENT_STATUS_REVIEW_OVERDUE
					notice = this.currentUserLatestReviewAssignment.status;
					break;
				}
			}

			// Incomplete submissions
			if (this.item.submissionProgress > 0) {
				notice = this.i18n.incompleteSubmissionNotice;
			}

			return notice;
		},

		/**
		 * Compile the count of open discussions
		 *
		 * @return int
		 */
		openQueryCount: function() {
			return _.where(this.activeStage.queries, {closed: false}).length;
		},

		/**
		 * Is this the review stage?
		 *
		 * @return bool
		 */
		isReviewStage: function() {
			return this.activeStage.id === 2 || this.activeStage.id === 3;
		},

		/**
		 * Retrieve the review assignments for the latest review round
		 *
		 * @return array
		 */
		currentReviewAssignments: function() {
			if (!this.item.reviewRounds.length || !this.item.reviewAssignments.length) {
				return [];
			}
			var currentReviewRoundId = this.item.reviewRounds[this.item.reviewRounds.length - 1].id;
			return _.filter(this.item.reviewAssignments, function(assignment) {
				return assignment.roundId === currentReviewRoundId;
			});
		},

		/**
		 * The current user's latest review assignment. This retrieves the
		 * review assignment from the latest round if available, or any other
		 * round if not available.
		 *
		 * @return object|false False if no review assignment exists
		 */
		currentUserLatestReviewAssignment: function() {

			if (!this.currentUserIsReviewer) {
				return false;
			}

			var assignments = _.where(this.item.reviewAssignments, {isCurrentUserAssigned: true});

			if (!assignments.length) {
				return false;
			}

			var latest = _.max(assignments, function(assignment) {
				return assignment.round;
			});

			switch (latest.statusId) {

				case 0: // REVIEW_ASSIGNMENT_STATUS_AWAITING_RESPONSE
				case 4: // REVIEW_ASSIGNMENT_STATUS_RESPONSE_OVERDUE
					latest.responsePending = true;
					latest.reviewPending = true;
					break;

				case 5: // REVIEW_ASSIGNMENT_STATUS_ACCEPTED
				case 6: // REVIEW_ASSIGNMENT_STATUS_REVIEW_OVERDUE
					latest.reviewPending = true;
					break;

				case 7: // REVIEW_ASSIGNMENT_STATUS_RECEIVED
				case 8: // REVIEW_ASSIGNMENT_STATUS_COMPLETE
				case 9: // REVIEW_ASSIGNMENT_STATUS_THANKED
					latest.reviewComplete = true;
					break;
			}

			return latest;
		},

		/**
		 * Compile the count of completed reviews
		 *
		 * @return int
		 */
		completedReviewsCount: function() {
			if (!this.isReviewStage) {
				return 0;
			}
			return _.filter(this.currentReviewAssignments, function(review) {
				return review.statusId >= 7; // REVIEW_ASSIGNMENT_STATUS_RECEIVED and above
			}).length;
		},

		/**
		 * Return a class to toggle the item mask
		 *
		 * @return string
		 */
		classMask: function() {
			if (!this.mask) {
				return '';
			} else if (this.mask === 'finish') {
				return '--finish';
			}
			var classes = ['--active'];
			if (this.mask === 'confirmingDelete' || this.mask === 'deleting') {
				classes.push('--alert');
			}

			return classes.join(' ');
		},

		/**
		 * ID attribute to use in aria-labelledby linking the reponse due date
		 * with it's label
		 *
		 * @return string
		 */
		responseDueLabelId: function() {
			return 'responseDueLabel' + this._uid;
		},

		/**
		 * ID attribute to use in aria-labelledby linking the review due date
		 * with it's label
		 *
		 * @return string
		 */
		reviewDueLabelId: function() {
			return 'reviewDueLabel' + this._uid;
		},

		/**
		 * ID attribute to use in aria-labelledby linking the reviews completed
		 * icons with their label
		 *
		 * @return string
		 */
		reviewsCompletedLabelId: function() {
			return 'reviewsCompletedLabel' + this._uid;
		},

		/**
		 * ID attribute to use in aria-labelledby linking the files prepared
		 * icons with their label
		 *
		 * @return string
		 */
		filesPreparedLabelId: function() {
			return 'filesPreparedLabel' + this._uid;
		},

		/**
		 * ID attribute to use in aria-labelledby linking the discussion icons
		 * with their label
		 *
		 * @return string
		 */
		discussionsLabelId: function() {
			return 'discussionsLabel' + this._uid;
		},
	},
	methods: {

		/**
		 * Load a modal displaying history and notes of a submission
		 */
		openInfoCenter: function() {

			var opts = {
				title: this.item.title,
				url: this.infoUrl.replace('__id__', this.item.id),
				closeCallback: this.resetFocusInfoCenter,
			};

			$('<div id="' + $.pkp.classes.Helper.uuid() + '" ' +
					'class="pkp_modal pkpModalWrapper" tabindex="-1"></div>')
				.pkpHandler('$.pkp.controllers.modal.AjaxModalHandler', opts);
		},

		/**
		 * Reset the focus on the info center link when the modal has been
		 * closed. This is a callback function passed into ModalHandler.js
		 */
		resetFocusInfoCenter: function() {
			this.$el.querySelector('.pkpListPanelItem__openInfoCenter').focus();
		},

		/**
		 * Display a confirmation prompt before deleting a submission
		 */
		deleteSubmissionPrompt: function() {
			this.mask = 'confirmingDelete';
		},

		/**
		 * Send a request to delete the submission and handle the response
		 */
		deleteSubmission: function() {

			this.mask = 'deleting';

			var self = this;
			$.ajax({
				url: this.getApiUrl(this.apiPath + '/' + this.item.id),
				type: 'DELETE',
				error: this.ajaxErrorCallback,
				success: function(r) {
					self.mask = 'finish';
					// Allow time for the finished CSS transition to display
					setTimeout(function() {
						pkp.eventBus.$emit('submissionDeleted', { id: self.item.id });
						self.cancelDeleteRequest();
					}, 300);
				},
				complete: function(r) {
					// Reset the mask in case there is an error
					if (self.mask === 'deleting') {
						self.cancelDeleteRequest();
					}
				}
			});
		},

		/**
		 * Cancel the delete request
		 */
		cancelDeleteRequest: function() {
			this.mask = null;
		},
	},
};
</script>
